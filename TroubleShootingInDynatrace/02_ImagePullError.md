**complete, deep-dive troubleshooting guide for Kubernetes image-pull failures from a private registry**, especially when *your cluster domain is whitelisted but image pull STILL fails*.
This is extremely common with **Dynatrace**, **AKS**, **EKS**, **GKE**, and **on-prem clusters**.

Let’s go step by step.

---

# ⭐ 1. Confirm the Actual Error Message

Run:

```bash
kubectl describe pod <pod>
```

Look for errors like:

```
Failed to pull image
Failed to pull and unpack image
Connection refused
dial tcp: i/o timeout
unauthorized: authentication required
x509: certificate signed by unknown authority
error parsing HTTP 403 response body
```

Each error type points to a different cause.

If you paste your **pod describe output**, I will pinpoint the exact fix.

---

# ⭐ 2. Common Reasons Even When “Cluster Domain Is Whitelisted”

Whitelisting your cluster domain **is not enough**.

Private registries check:

| Check                        | Meaning                         | Status          |
| ---------------------------- | ------------------------------- | --------------- |
| ✔ IP-based whitelist         | Node’s public IP allowed?       | Often missing   |
| ✔ DNS whitelist              | FQDN cluster name               | Already done    |
| ✔ NAT gateway IP             | Nodes behind NAT?               | Usually missing |
| ✔ Container runtime identity | containerd needs pull secret    | Missing         |
| ✔ TLS certificate trust      | Root CA installed?              | Missing         |
| ✔ Proxy settings             | HTTP_PROXY, HTTPS_PROXY         | Missing         |
| ✔ HARSH firewalls            | Block port 443 or registry path | Hidden issue    |

In most cases, the registry is expecting **Node IPs**, NOT the Kubernetes API server domain.

---

# ⭐ 3. Confirm You Are Pulling FROM Node, Not From the Cluster

Kubernetes nodes pull the image, not the API server.

Meaning:

**Your worker node(s) must be whitelisted**, not cluster domain.

Check node public IPs:

```bash
kubectl get nodes -o wide
```

Then whitelist:

* Node public IPv4
* Node NAT IP
* Cloud provider outbound IP
* Cluster egress IP (AKS)
* VPC egress IP (EKS)

---

# ⭐ 4. Check if You Added a Proper ImagePullSecret

### Step 1: Create secret

```bash
kubectl create secret docker-registry mysecret \
  --docker-server=<registry-server> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>
```

### Step 2: Add to Pod/Deployment

```yaml
imagePullSecrets:
  - name: mysecret
```

### Step 3: Check if secret is applied

```bash
kubectl get pods <pod> -o jsonpath='{.spec.imagePullSecrets}'
```

If empty → secret not attached.

---

# ⭐ 5. Check Container Runtime Connectivity (MOST COMMON)

Log into a node:

### For SSH-enabled clusters:

```
ssh <node>
```

Then test:

```
crictl pull <registry>/<image>:tag
```

If it fails, registry whitelist is incorrect.

---

# ⭐ 6. Check DNS Resolution from Node

Inside node:

```
nslookup <private-registry-name>
curl -v https://<private-registry-name>
```

If DNS fails → image will NOT pull.

---

# ⭐ 7. Check Registry SSL Certificate Issues

If you see:

```
x509: certificate signed by unknown authority
```

Fix:

### Option 1: Add registry CA to nodes

Place CA into:

* `/etc/ssl/certs/`
* `/usr/local/share/ca-certificates/`

Then run:

```
update-ca-certificates
systemctl restart containerd
```

### Option 2: Add insecure registry (temporary only!)

In:

```
/etc/containerd/config.toml
```

Add:

```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."myregistry.com"]
  endpoint = ["http://myregistry.com"]
```

Restart:

```
systemctl restart containerd
```

---

# ⭐ 8. Check Firewall / Proxy Blocking the Registry

From a node:

```
curl -vk https://<registry-name>/v2/
```

Expected response:

```
401 Unauthorized
```

If you get timeout → firewall/network issue.

If you get 403 → whitelist failure.

If you get 500 → registry-side issue.

---

# ⭐ 9. Check the Private Registry Has the Correct Repository Path

The registry may require:

```
myregistry.com/project/image:tag
```

Instead of:

```
myregistry.com/image:tag
```

Try:

```
skopeo inspect docker://myregistry.com/.../image
```

---

# ⭐ 10. Most Common Cluster Scenarios

---

## ⭐ **AKS (Azure)**

Your outbound IP = **Azure NAT gateway** or **load balancer outbound IP**.

Find it:

```bash
az network public-ip list --resource-group <rg>
```

Whitelist that instead of cluster FQDN.

---

## ⭐ **EKS (AWS)**

Nodes use the **VPC NAT Gateway IP** for outbound traffic.

Find NAT IP:

```bash
aws ec2 describe-nat-gateways --region <region>
```

Whitelist the NAT IP.

---

## ⭐ **GKE (Google)**

Outbound IP depends on:

* Cloud NAT
* VPC routing

Find it:

```bash
gcloud compute addresses list
```

Then whitelist that.

---

## ⭐ **Bare Metal / On-Prem**

Workers use:

* Node IP
* NAT IP
* Firewall outbound IP

Whitelist all egress interfaces:

```
ip route
ip addr
```

---

# ⭐ 11. Check Dynatrace-Injected Pods Specifically

If image pulling only fails **after Dynatrace injection**:

### Possible root causes:

❌ missing proxy settings
❌ OneAgent init container needs proxy but not configured
❌ containerd blocked by ACL
❌ init container image cannot pull

Check init container status:

```bash
kubectl describe pod <pod> | grep -i init
```

Check logs:

```bash
kubectl logs <pod> -c dynatrace-oneagent-init
```

---

