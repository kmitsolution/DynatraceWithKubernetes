
1. Create a Dynatrace account / get **Environment ID** and two tokens (API token with `Installer/Token` and a **PAAS** token). (Dynatrace UI gives these.) ([Dynatrace Documentation][1])
2. Prepare Kubernetes: `kubectl` access, a `dynatrace` namespace and a secret holding tokens.
3. Install the **Dynatrace Operator** (preferred) — either via the install command provided by the Dynatrace UI or via the operator YAML / Helm. The Operator manages OneAgent and ActiveGate. ([Dynatrace Documentation][2])
4. Create a `DynaKube` (custom resource) or OneAgent CR to enable OneAgent DaemonSet / Operator-managed rollout. Verify pods. ([Dynatrace Documentation][3])
5. (If needed) Deploy ActiveGate (container/statefulset or VM) for API routing, Kubernetes API monitoring, or multi-cluster setups. ([Dynatrace Documentation][4])
6. Validate in the Dynatrace UI (Kubernetes dashboard, metrics, traces, logs), then iterate (tagging, filtering, RBAC). ([Dynatrace Documentation][5])

---

# Step-by-step with commands and templates

## A. Prep: tokens & cluster access

1. In Dynatrace UI → **Deploy Dynatrace** → **Start installation** → **Kubernetes**: follow the wizard to create tokens and get the recommended install command. (It generates the operator install command tailored to your environment.) ([Dynatrace Documentation][2])

2. Locally, ensure you have cluster admin `kubectl` access for the target cluster:

```bash
kubectl version --client && kubectl cluster-info
```

## B. Create namespace & secret (replace placeholders)

Create a namespace and a Kubernetes secret containing your **apiToken** and **paasToken** (do not paste real tokens into public chat):

```bash
kubectl create namespace dynatrace

kubectl -n dynatrace create secret generic dynatrace \
  --from-literal="apiToken=<YOUR_API_TOKEN>" \
  --from-literal="paasToken=<YOUR_PAAS_TOKEN>"
```

This secret name is what you will reference in the Operator/DynaKube configuration. ([Dynatrace Documentation][3])

## C. Install Dynatrace Operator

Option A — **Use the install command from Dynatrace UI** (recommended). The UI will give you a `kubectl apply -f -` command that wires in your environment automatically. ([Dynatrace Documentation][2])

Option B — **Manual** (Operator GitHub / prebuilt YAML):

```bash
# example — get operator YAML from GitHub and apply (change tag/version as needed)
kubectl apply -f https://raw.githubusercontent.com/Dynatrace/dynatrace-operator/main/deploy/kubernetes.yaml -n dynatrace
```

(Operator repo: contains examples and CRD definitions.) ([GitHub][6])

Wait for the operator pods to show `Running`:

```bash
kubectl -n dynatrace get pods
```

## D. Create DynaKube (the CR) — basic template

Below is a **minimal** example to adapt. *Do not forget to replace* the `environment` / `apiUrl` / `secretName` placeholders with your actual values:

```yaml
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  apiUrl: "https://{your-environment-id}.live.dynatrace.com"   # replace
  tokens:
    # name of the secret you created
    secretName: dynatrace
  oneAgent:
    classicFullStack: {}       # or use cloudNativeFullStack/hostMonitoring depending on needs
  activeGate:
    enabled: true
    # additional configuration options exist for routing, kubernetes-monitoring, etc.
```

Apply it:

```bash
kubectl apply -f dynakube.yaml
kubectl -n dynatrace get pods -l app.kubernetes.io/name=dynatrace-oneagent
```

Notes:

* The Operator can create a DaemonSet of OneAgent on each node or inject OneAgent into pods. Choose the mode that fits your cluster/app model. ([Dynatrace Documentation][3])
* If you prefer, the OneAgent operator Helm chart is available as well — useful for automation. ([Artifact Hub][7])

## E. ActiveGate (when & how)

* If you need Kubernetes API monitoring, custom routing, or a network proxy between Dynatrace and your cluster, deploy ActiveGate as a container (StatefulSet) or VM. The Operator can manage ActiveGate capabilities; you can also manually deploy an ActiveGate StatefulSet. ([Dynatrace Documentation][4])

## F. Verify & explore

1. In Dynatrace UI: navigate to **Kubernetes** / **Cluster** view — you should see nodes, namespaces, pods and basic metrics within minutes. ([Dynatrace Documentation][1])
2. Use `kubectl get pods --all-namespaces` to confirm OneAgent & ActiveGate pod status.
3. Check logs if pods crash: `kubectl -n dynatrace logs <pod-name>`.

---

