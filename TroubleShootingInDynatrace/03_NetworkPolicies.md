**Network Policies in Kubernetes**, and **how to use them safely when Dynatrace is installed** (OneAgent, ActiveGate, Operator, CSI driver, webhook, etc.).

This will help you understand:

✔ What NetworkPolicies do
✔ How they interact with Dynatrace
✔ Traffic Dynatrace needs to function
✔ Example NetworkPolicies for Dynatrace
✔ How to avoid breaking monitoring

Let’s go step by step.

---

# ⭐ 1. What Are Network Policies?

NetworkPolicies in Kubernetes control **which pods can talk to which pods** at **L3/L4** (IP, port).

They allow you to enforce:

* **Ingress rules** → Who is allowed to talk *to* a pod
* **Egress rules** → Who the pod is allowed to talk *to*

Think of them like a **firewall** for pods.

Without NetworkPolicies:

> All pods can talk to all pods.

With NetworkPolicies:

> Only allowed flows can communicate.

---

# ⭐ 2. How NetworkPolicies Affect Dynatrace

Dynatrace requires **network communication** for:

### ✔ Telemetry upload

Pods with OneAgent must send data to:

* Dynatrace SaaS endpoint
* or ActiveGate

### ✔ Downloading OneAgent binaries

CSI driver + init containers must download modules from:

```
<tenant>.dynatrace.com
```

### ✔ Webhook communication

Kubernetes API → dynatrace-webhook service (port 443)

### ✔ ActiveGate communications

ActiveGate needs to reach:

* Dynatrace SaaS
* Kubernetes API
* Cluster workloads (optional)

### ✔ Cluster scraping

ActiveGate must reach:

* Kube API server
* Metrics endpoints
* Pods / services (if configured)

If NetworkPolicies are too strict → Dynatrace breaks.

---

# ⭐ 3. What Traffic Dynatrace Needs

Below is the **minimum allowed traffic**.

---

# 🔹 **A. Pod → Dynatrace SaaS / Managed**

For OneAgent:

```
443/TCP
*.dynatrace.com
```

OR if you use ActiveGate:

```
Pod → ActiveGate (HTTP/TLS)
```

---

# 🔹 **B. Pod → dynatrace-webhook**

For injection:

```
Pod creation → dynatrace-webhook:443
```

Webhook must always be allowed.

---

# 🔹 **C. CSI Driver → Dynatrace SaaS**

To download agent binaries:

```
csi-nodeplugin → dynatrace SaaS:443
csi-controller → dynatrace SaaS:443
```

---

# 🔹 **D. ActiveGate → Dynatrace SaaS**

```
ActiveGate → <tenant>.dynatrace.com:443
```

---

# 🔹 **E. ActiveGate → Kube API Server**

For cluster discovery:

```
ActiveGate → https://kubernetes.default.svc:443
```

---

# ⭐ 4. Sample Network Policy for Dynatrace Operator

Allow the operator + webhook:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dynatrace-operator-allow
  namespace: dynatrace
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: dynatrace-operator
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector: {}  # webhook from API server
    ports:
    - port: 443
      protocol: TCP
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - port: 443
      protocol: TCP
```

---

# ⭐ 5. Sample Network Policy for Dynatrace Webhook

Allow Kubernetes API → webhook:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dynatrace-webhook-allow
  namespace: dynatrace
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: dynatrace-webhook
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {} # kube-apiserver IPs
    ports:
    - port: 443
      protocol: TCP
```

---

# ⭐ 6. Sample Network Policy for Monitoring Injection (Workloads)

Allow workloads to reach Dynatrace SaaS or ActiveGate:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dynatrace-egress
spec:
  podSelector: {}   # all pods, or restrict by label
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443
```

This is REQUIRED for:

* OneAgent to send monitoring data
* initContainer to download agent code

If not allowed → pods break OR OneAgent does not start.

---

# ⭐ 7. Sample Network Policy for CSI Driver

CSI Node Plugin needs outbound internet:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-csi-egress
  namespace: dynatrace
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: dynatrace-oneagent-csi-node
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - port: 443
      protocol: TCP
```

---

# ⭐ 8. Sample Network Policy for ActiveGate

ActiveGate needs to reach:

* Dynatrace SaaS (443)
* Kubernetes API server (443)
* Workload metrics endpoints (kubelet or scrape targets)

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-activegate
  namespace: dynatrace
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: dynatrace-activegate
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock: { cidr: 0.0.0.0/0 }
      ports:
      - { port: 443, protocol: TCP }
```

---

# ⭐ 9. How to Detect NetworkPolicy Blocking Dynatrace

### ✔ Check pod describe:

```
kubectl describe pod <pod> | grep -i timeout
```

### ✔ Look for code module errors:

```
OneAgent init: failed to reach server
```

### ✔ Test outbound inside pod:

```
curl -vk https://<tenant>.live.dynatrace.com
```

### ✔ Check logs:

Operator:

```
kubectl logs deploy/dynatrace-operator -n dynatrace
```

CSI:

```
kubectl logs ds/dynatrace-oneagent-csi-node -n dynatrace
```

ActiveGate:

```
kubectl logs sts/dynatrace-activegate -n dynatrace
```

If you see “connection refused” → **NetworkPolicy or firewall**.

---

# ⭐ 10. Summary — What NetworkPolicies Must Allow For Dynatrace

### 🔵 REQUIRED

* Pods → ActiveGate or SaaS
* Init container → SaaS
* CSI → SaaS
* Webhook → Kube API
* ActiveGate → Kube API
* ActiveGate → internet

### 🔵 OPTIONAL

* Pod-to-pod monitoring
* External service monitoring
* Service mesh visibility

### 🔴 MUST NEVER BLOCK

* port 443 outbound for any Dynatrace component
* dynatrace-webhook inbound from API server

If blocked → **injection breaks**.

---

# ⭐ If you want next topic:

I can explain:

### ✔ “How to write namespace-level network policies for Dynatrace”

### ✔ “How to test network connectivity in pods with busybox”

### ✔ “NetworkPolicy vs CNI vs firewall vs proxy — what affects Dynatrace?”

### ✔ “How Dynatrace auto-detects network issues and slow calls”

Just say **“next: <topic>”**.
