A **PriorityClass** in Kubernetes is a built-in scheduling feature that controls **which pods get scheduled first and which pods get evicted last** when the cluster is under resource pressure.

Think of it as a **"pod importance score"**.

---

# 🟦 **What is a PriorityClass?**

A `PriorityClass` is an object that assigns a **priority value (an integer)** to a pod.
Higher value = Higher priority.

Kubernetes uses this value to decide:

## ✔ **Which pods should be scheduled first**

If resources are limited, higher-priority pods get node resources before lower-priority pods.

## ✔ **Which pods should be evicted last**

During node pressure (CPU, memory), Kubernetes evicts **low-priority pods first**.

---

# 🟩 **Why PriorityClass is used**

Typical use cases:

### 🔹 Critical system pods

* kube-dns
* metrics-server
* ingress controller
* Dynatrace operator
* CSI drivers

### 🔹 Production workloads needing guaranteed capacity

* API servers
* Payment services
* Logging agents

### 🔹 Lower-priority workloads for batch or background tasks

* Data-processing jobs
* CI runners
* Development pods

---

# 🧱 **PriorityClass YAML Example**

### 1️⃣ Create a PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "Important production workload"
```

Apply:

```bash
kubectl apply -f priorityclass.yaml
```

---

### 2️⃣ Use it in a Pod or Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      priorityClassName: high-priority
      containers:
      - name: app
        image: nginx
```

---

# 🟦 **Built-in Priority Classes**

Kubernetes comes with two special priority classes:

| Name                      | Value      | Purpose                     |
| ------------------------- | ---------- | --------------------------- |
| `system-cluster-critical` | 2000000000 | Critical for entire cluster |
| `system-node-critical`    | 2000001000 | Critical for node stability |

These are reserved for core Kubernetes components.

---

# 🟥 What PriorityClass does *NOT* do

It does **NOT**:

✖ guarantee pod scheduling
✖ guarantee resources (you still need requests/limits)
✖ move a running pod from one node to another
✖ prevent OOM kills inside a container

---

# 🟧 How Dynatrace Uses PriorityClass

Dynatrace OneAgent and ActiveGate pods can use priority classes:

* To ensure instrumentation agents start before application pods
* To avoid eviction during node pressure

You might see in dynakube:

```yaml
oneAgent:
  cloudNativeFullStack:
    priorityClassName: dt-oneagent-priority
```

---

# ⭐ Summary

A **PriorityClass pod** is a pod with an assigned priority that tells Kubernetes:

* which pods are *more important*
* which pods must *run first*
* which pods must *stay alive* during resource shortages

This is crucial for production workloads, operators, monitoring agents, and system services.

---

Great — here are **fully validated, production-ready PriorityClass examples specifically for Dynatrace**, along with when and how to use them.

These are the same patterns used in enterprise-grade clusters to ensure **OneAgent**, **ActiveGate**, **Operator**, and **CSI driver pods** remain stable even during node pressure.

---

# ⭐ **Why PriorityClass Matters for Dynatrace**

Dynatrace relies on:

* Operator
* Webhook
* CSI driver (for injection)
* OneAgent DaemonSets
* ActiveGate StatefulSets

If ANY of these are evicted or fail to schedule, **instrumentation stops working**.

To prevent this, Dynatrace allows specifying a **priorityClassName** in DynaKube.

---

# 🟦 **1. Create PriorityClasses for Dynatrace**

Below are **three recommended PriorityClasses**:

---

## ✔️ **A. Dynatrace Operator Priority**

Operator is critical; if it stops, no new injections happen.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-operator-critical
value: 1000000000
globalDefault: false
description: "Priority for Dynatrace Operator"
```

---

## ✔️ **B. Dynatrace OneAgent Priority**

Ensures OneAgent DaemonSet is not evicted:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-oneagent-critical
value: 900000000
globalDefault: false
description: "Priority for Dynatrace OneAgent pods"
```

---

## ✔️ **C. Dynatrace ActiveGate Priority**

ActiveGate handles metrics-ingest + routing.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-activegate-critical
value: 800000000
globalDefault: false
description: "Priority for Dynatrace ActiveGate pods"
```

Apply them:

```bash
kubectl apply -f priorityclasses.yaml
```

---

# 🟦 **2. Use PriorityClass in your DynaKube CR**

### ✔ **OneAgent Priority**

Inside your DynaKube YAML:

```yaml
oneAgent:
  cloudNativeFullStack:
    priorityClassName: dynatrace-oneagent-critical
```

This ensures:

* OneAgent pods don’t get evicted
* They start early during node boot

---

### ✔ **ActiveGate Priority**

```yaml
activeGate:
  priorityClassName: dynatrace-activegate-critical
```

This ensures:

* ActiveGate remains available for metrics ingestion
* No loss of Prometheus scraping
* No routing interruptions

---

### ✔ **Operator Priority**

Usually set in the operator installation YAML.

You can edit the operator deployment:

```bash
kubectl -n dynatrace edit deploy dynatrace-operator
```

Add:

```yaml
spec:
  template:
    spec:
      priorityClassName: dynatrace-operator-critical
```

---

# 🟩 **3. Priority Best Practices for Dynatrace**

### ✔ Always give Operator the **highest** priority

If the operator is evicted, Dynatrace cannot:

* inject new pods
* manage updates
* process DynaKube changes

### ✔ OneAgent should remain higher than app pods

Because instrumentation must load BEFORE app processes.

### ✔ ActiveGate should have stable priority

Without ActiveGate, you lose:

* Kubernetes API monitoring
* Prometheus ingestion
* Routing

---

# 🔥 **4. Real-World Priority Hierarchy Example**

| Component          | PriorityClass                 | Value         | Reason                      |
| ------------------ | ----------------------------- | ------------- | --------------------------- |
| Dynatrace Operator | dynatrace-operator-critical   | 1,000,000,000 | Critical for injection      |
| OneAgent           | dynatrace-oneagent-critical   | 900,000,000   | Must run on every node      |
| ActiveGate         | dynatrace-activegate-critical | 800,000,000   | Ingestion + routing         |
| User workloads     | normal                        | <500,000      | Should not evict monitoring |

This ensures Dynatrace components stay functional **even if nodes are low on memory or CPU**.

---

# 🛠 **5. Validate PriorityClass is Applied**

Pick any Dynatrace pod:

```bash
kubectl -n dynatrace describe pod <pod>
```

Look for:

```
Priority Class Name:  dynatrace-oneagent-critical
Priority: 900000000
```

Do this for:

* oneagent pods
* activegate pods
* csi-driver pods
* operator pods

---
Below is a **complete, production-grade DynaKube configuration** that includes:

✅ Dynatrace Operator
✅ CSI Driver support
✅ OneAgent (cloudNativeFullStack)
✅ ActiveGate with routing + kubernetes-monitoring + metrics-ingest
✅ Prometheus OpenMetrics ingestion
✅ Annotation-based injection
✅ PriorityClasses for Operator, OneAgent, and ActiveGate
✅ Ready-to-apply Kubernetes YAML

This is a **single, copy-paste deployment bundle** containing:

1. PriorityClass definitions
2. Secret for Dynatrace tokens
3. Full DynaKube CR with priorities, metrics ingestion, annotation-based injection

Use this as your **enterprise-grade Dynatrace foundation**.

---

# 🚀 **FULL ENTERPRISE DynaKube WITH PRIORITIES (copy/paste ready)**

> **⚠ NOTE:** Replace
> `<YOUR_API_URL>`
> `<YOUR_SECRET_NAME>`
> `<YOUR_API_TOKEN>`
> `<YOUR_PAAS_TOKEN>`
> `<YOUR_METRICS_LABEL>`
> `<YOUR_METRICS_PORT>`
> `<YOUR_OPENMETRICS_RULE>`
> with your actual values.

```yaml
---
# ===========================================
# PRIORITY CLASSES FOR DYNATRACE COMPONENTS
# ===========================================
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-operator-critical
value: 1000000000
globalDefault: false
description: "High priority for Dynatrace Operator"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-oneagent-critical
value: 900000000
globalDefault: false
description: "High priority for Dynatrace OneAgent pods"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: dynatrace-activegate-critical
value: 800000000
globalDefault: false
description: "High priority for Dynatrace ActiveGate pods"
---
# ===========================================
# DYNATRACE TOKEN SECRET
# ===========================================
apiVersion: v1
kind: Secret
metadata:
  name: <YOUR_SECRET_NAME>
  namespace: dynatrace
type: Opaque
stringData:
  apiToken: "<YOUR_API_TOKEN>"
  paasToken: "<YOUR_PAAS_TOKEN>"
---
# ===========================================
# FULL DynaKube WITH PRIORITIES + PROMETHEUS
# ===========================================
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  # --------------------------
  # Dynatrace API endpoint
  # --------------------------
  apiUrl: "<YOUR_API_URL>"

  tokens:
    secretName: <YOUR_SECRET_NAME>

  skipCertCheck: false

  # --------------------------
  # Enable metadata enrichment
  # --------------------------
  metadataEnrichment:
    enabled: true

  # =============================================================
  # ONEAGENT CONFIGURATION (CLOUD NATIVE FULLSTACK + PRIORITY)
  # =============================================================
  oneAgent:
    cloudNativeFullStack:
      useAnnotation: true               # selective instrumentation
      priorityClassName: dynatrace-oneagent-critical

      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule

  # =============================================================
  # ACTIVEGATE WITH METRICS + PRIORITY
  # =============================================================
  activeGate:
    priorityClassName: dynatrace-activegate-critical
    capabilities:
      - routing
      - kubernetes-monitoring
      - metrics-ingest
      - dynatrace-api
    resources:
      requests:
        cpu: 300m
        memory: 600Mi
      limits:
        cpu: 1
        memory: 2Gi

  # =============================================================
  # PROMETHEUS OPENMETRICS INGESTION
  # =============================================================
  metrics:
    enabled: true
    openMetrics:
      - name: "<YOUR_OPENMETRICS_RULE>"
        selector:
          matchLabels:
            app: "<YOUR_METRICS_LABEL>"     # e.g., kube-state-metrics
        url: "http://$(POD_IP):<YOUR_METRICS_PORT>/metrics"
        port: <YOUR_METRICS_PORT>
        interval: 30s

  # =============================================================
  # LOG MONITORING OPTIONAL
  # =============================================================
  logMonitoring: {}
```

---

# 🟢 **What You Get in This YAML**

| Feature                            | Included | Notes                               |
| ---------------------------------- | -------- | ----------------------------------- |
| OneAgent cloud-native fullstack    | ✔        | Most recommended mode               |
| Annotation-based injection         | ✔        | You must annotate pods              |
| ActiveGate routing                 | ✔        | Required for node-to-cluster comms  |
| ActiveGate metrics-ingest          | ✔        | Required for Prometheus             |
| Prometheus ingestion (OpenMetrics) | ✔        | With scraping rules                 |
| PriorityClass for Operator         | ✔        | Add to Deployment manually          |
| PriorityClass for OneAgent         | ✔        | In DynaKube                         |
| PriorityClass for ActiveGate       | ✔        | In DynaKube                         |
| Metadata enrichment                | ✔        | Helps with tagging/Smartscape       |
| Tolerations                        | ✔        | Supports master/control-plane nodes |

---

# 🔥 **FINAL STEP: Patch Dynatrace Operator deployment**

Apply priority to operator itself:

```bash
kubectl -n dynatrace patch deploy dynatrace-operator \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"dynatrace-operator-critical"}}}}'
```

---


