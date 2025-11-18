---

## 📝 What the `kubernetes-csi.yaml` typically contains

For the Dynatrace Operator installation with CSI (= code module injection via CSI), the YAML manifest usually includes:

1. Namespace creation (e.g., `dynatrace`)
2. ServiceAccount, Roles, RoleBindings for operator, webhook, CSI driver
3. CRD definitions (CustomResourceDefinitions) for `DynaKube` etc.
4. Operator Deployment (`dynatrace-operator`)
5. Webhook Deployment / Service (`dynatrace-webhook`)
6. CSI Driver DaemonSet(s) / Controller for oneagent-csi
7. CSI Driver CRDs (CSIDriver)
8. MutatingWebhookConfiguration / ValidatingWebhookConfiguration for injection
9. Possibly ActiveGate StatefulSet including CSI volumes
10. ConfigMaps / Secrets for default configs

---

## 🔍 Explanation of major sections you will find (and what they do)

Below is a breakdown of typical sections and their functions:

| Section                                                           | Purpose                                                                                                                                                                                                                                                 |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Namespace**                                                     | A dedicated namespace (often `dynatrace`) is created to isolate operator & components.                                                                                                                                                                  |
| **ServiceAccount / RBAC**                                         | Grants operator, webhook, CSI drivers permissions to watch pods, create volumes, modify namespaces, etc.                                                                                                                                                |
| **CRD definitions**                                               | Defines custom resource kinds like `DynaKube`, `OneAgent`, `ActiveGate`, enabling you to use those types.                                                                                                                                               |
| **Operator Deployment**                                           | The main controller that watches `DynaKube` CRs and manages rollout of OneAgent, ActiveGate, CSI driver etc. ([Dynatrace Documentation][1])                                                                                                             |
| **Webhook Deployment + Service**                                  | This mutates Kubernetes pods (when annotation/namespace selection matches) to inject the necessary volumes & init containers for OneAgent + CSI driver. Also validates CRs.                                                                             |
| **CSI Driver Components**                                         | Includes CSIDriver resource, driver DaemonSet, Node plugin, Controller plugin. The CSI driver handles the *volume mounting* of the agent code modules into pods. This is crucial for the “cloud native full-stack” mode. ([Dynatrace Documentation][2]) |
| **MutatingWebhookConfiguration / ValidatingWebhookConfiguration** | Registers the webhook with Kubernetes API server so pod mutations or CR validations happen at creation/update time.                                                                                                                                     |
| **ConfigMap / Default values**                                    | Some initial configuration defaults may be placed here for the operator/CSI driver.                                                                                                                                                                     |
| **Secret template / Example**                                     | Some manifests include a placeholder secret template (or instructions) where you supply your `apiToken`, etc.                                                                                                                                           |
| **ActiveGate StatefulSet / Service**                              | In some manifests the ActiveGate is included so that routing, metrics-ingest, kubernetes-monitoring are available. With CSI enabled you might see related volume mounts for ActiveGate.                                                                 |

---

## ✅ What to Look Out For / What to Understand

* **CSIDriver resource**: This registers the driver with Kubernetes so that volumes can be provisioned by the driver.
* **DaemonSet vs Deployment**:

  * The CSI Node plugin runs as a DaemonSet (one per node) so every node can mount volumes.
  * The operator runs as a Deployment (typically single replica + leader election).
* **Pod mutation**: When you deploy an application Pod and it matches injection criteria (annotation, namespaceSelector), the webhook intercepts it and injects:

  * a volume referencing the CSI driver
  * an init container to load agent modules
  * environment variables / mounts needed by OneAgent
* **PriorityClass / Node affinity / Taints & tolerations**: For high-priority components (operator/ActiveGate) you may see priorityClassName or tolerations so they run even on control-plane nodes or survive node pressure.
* **Volume mount paths**: For code modules you’ll see mounts like `/opt/dynatrace/oneagent` inside pods, which come from the CSI volume.
* **Versioning and updates**: The manifest may include image tags for operator/webhook/CSI driver, so you can control version upgrades.

---

## 🔮 What We *Don’t* See (usually)

* Specific configuration for *your* DynaKube (that comes later)
* Application-specific monitoring config
* Secrets with real tokens (you create those)
* Custom Prometheus scraping rules (those go in DynaKube CR)

---



[1]: https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/how-it-works/components/dynatrace-operator?utm_source=chatgpt.com "Dynatrace Operator"
[2]: https://docs.dynatrace.com/docs/ingest-from/setup-on-k8s/guides/migration/classic-to-cloud-native?utm_source=chatgpt.com "Migrate from classic full-stack to cloud-native full-stack mode"
