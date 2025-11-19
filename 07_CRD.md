
#  PART 1 — What is a CRD in Kubernetes? (Simple Explanation)

A **CRD (Custom Resource Definition)** allows you to *extend Kubernetes* by adding your **own resource types**, the same way Kubernetes has built-in types like:

* Pod
* Service
* Deployment
* Namespace

A CRD lets you create **new resource types**, for example:

* `Database`
* `BackupJob`
* `Certificate`
* `RedisCluster`

These become **first-class API objects** inside Kubernetes.

---

# ⭐ PART 2 — CRD Simple Kubernetes Example

Let’s create a simple CRD called **Fruit**.

## 🥭 1. Define the CRD

`fruit-crd.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: fruits.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: fruits
    singular: fruit
    kind: Fruit
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                color:
                  type: string
                taste:
                  type: string
```

Apply:

```bash
kubectl apply -f fruit-crd.yaml
```

Now Kubernetes understands a **new API kind**:

```
Fruit
```

---

## 🥭 2. Create a Fruit (Custom Resource)

```yaml
apiVersion: example.com/v1
kind: Fruit
metadata:
  name: mango
spec:
  color: yellow
  taste: sweet
```

Apply:

```bash
kubectl apply -f mango.yaml
```

Check:

```bash
kubectl get fruits
kubectl describe fruit mango
```

Just like Pods/Deployments, **fruits** are now real Kubernetes objects.

---

# ⭐ PART 3 — Why CRDs Exist

CRDs allow tools and platforms to:

✔ Extend Kubernetes
✔ Add new APIs
✔ Build operators
✔ Manage custom infrastructure

This is the foundation for:

* Prometheus Operator
* Cert-Manager
* ArgoCD
* Istio
* Dynatrace Operator

---

# ⭐ PART 4 — Dynatrace CRD Explanation

Dynatrace uses CRDs to manage all the components it deploys.

The main Dynatrace CRDs include:

| CRD            | Description                                                                                                             |
| -------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **DynaKube**   | The main configuration object that describes how Dynatrace installs OneAgent, ActiveGate, metrics ingestion, logs, etc. |
| **OneAgent**   | Legacy configuration, replaced by DynaKube                                                                              |
| **ActiveGate** | Defines ActiveGate configuration                                                                                        |
| **DataIngest** | For custom metric ingestion options                                                                                     |

The Dynatrace Operator watches these CRDs and acts on them.

---

# ⭐ PART 5 — Dynatrace CRD (Real Example)

Whenever you apply a **DynaKube** resource like this:

```yaml
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
spec:
  apiUrl: https://abc123.live.dynatrace.com/api
  oneAgent:
    cloudNativeFullStack: {}
  activeGate:
    capabilities:
      - routing
```

You are **not applying a Pod or Deployment**.

You are applying a **Custom Resource**, defined by this CRD:

```yaml
kind: CustomResourceDefinition
metadata:
  name: dynakubes.dynatrace.com
spec:
  group: dynatrace.com
  names:
    plural: dynakubes
    singular: dynakube
    kind: DynaKube
  scope: Namespaced
  versions:
    - name: v1beta1
      served: true
      storage: true
```

This CRD tells Kubernetes how to understand:

```
kind: DynaKube
```

---

# ⭐ PART 6 — How Dynatrace Operator Uses CRDs

Once the CRD exists, you create **Custom Resources (CRs)**.

### Example CR:

```yaml
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
```

The Operator sees this CR and performs actions:

### 🔧 Operator Actions:

* Deploys OneAgent DaemonSet
* Deploys ActiveGate StatefulSet
* Deploys CSI driver
* Creates webhook
* Manages updates
* Scrapes Prometheus metrics
* Enables log ingestion
* Injects code modules

This is the **Operator pattern**:

> CRDs add new object types → Operator watches CRDs → Operator acts to create resources.

---

# ⭐ PART 7 — Simple Summary

### ✔ CRD = the **definition** of a new Kubernetes object

### ✔ CR = an **instance** of that object

### ✔ Operator = a controller that **manages** those CRs

---

# 🎯 Final Comparison

| Concept                | Simple Example                | Dynatrace Example   |
| ---------------------- | ----------------------------- | ------------------- |
| CRD defines a new kind | Fruit                         | DynaKube            |
| CR is an instance      | mango (Fruit)                 | dynakube (DynaKube) |
| Operator manages CR    | Fruit Operator (hypothetical) | Dynatrace Operator  |

Great — this is a **CustomResourceDefinition (CRD)** for **DynaKube**, which is the core configuration object used by the **Dynatrace Operator** to deploy OneAgent, ActiveGate, CSI driver, and Prometheus ingestion into Kubernetes.

I’ll break this down into **very clear sections**, so you understand:

* What each part does
* Why Dynatrace needs it
* How Kubernetes uses it internally

---

# ⭐ **Section 1 — apiVersion + kind**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
```

This tells Kubernetes:

> “I am defining a NEW API type.”

This CRD adds **DynaKube** as a first-class Kubernetes resource — just like Pods, Services, Deployments.

---

# ⭐ **Section 2 — metadata**

```yaml
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.18.0
  name: dynakubes.dynatrace.com
```

### Meaning:

* This CRD was generated using **Kubebuilder** (a framework used to build operators).
* The CRD name must be:

  ```
  <plural>.<group>
  ```

  → `dynakubes.dynatrace.com`

---

# ⭐ **Section 3 — spec.conversion**

```yaml
spec:
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          name: dynatrace-webhook
          namespace: dynatrace
          path: /convert
      conversionReviewVersions:
      - v1
      - v1beta1
```

### What this means:

Dynatrace supports **multiple API versions** of the DynaKube CR (e.g., `v1beta1`, `v1alpha1`, `v1`).
When users apply older YAML, the Operator automatically **converts** it to the internal version using a webhook.

✔ This webhook is hosted by:
**Service:** `dynatrace-webhook`
**Namespace:** `dynatrace`
**Path:** `/convert`

This ensures backwards compatibility.

---

# ⭐ **Section 4 — group + names**

```yaml
group: dynatrace.com
names:
  categories:
  - dynatrace
  kind: DynaKube
  listKind: DynaKubeList
  plural: dynakubes
  shortNames:
  - dk
  - dks
  singular: dynakube
scope: Namespaced
```

### This defines how Kubernetes exposes this API:

* **group**: `dynatrace.com`
* **kind**: `DynaKube` (like kind: Pod)
* **plural**: `dynakubes`
* **shortNames**: you can run:

  ```bash
  kubectl get dk
  ```
* **scope:** Namespaced
  → each DynaKube lives inside a specific namespace

---

# ⭐ **Section 5 — versions list**

This part defines all supported API versions of DynaKube.

```yaml
versions:
- name: v1beta1
  deprecated: true
```

This means:

* v1beta1 is currently supported
* But will be **removed** in a future release

It includes warnings to notify users.

---

# ⭐ **Section 6 — additionalPrinterColumns**

```yaml
additionalPrinterColumns:
- jsonPath: .spec.apiUrl
  name: ApiUrl
  type: string
```

This defines what gets shown when you run:

```bash
kubectl get dynakubes
```

You might see columns like:

```
NAME       APIURL                         STATUS       AGE
dynakube   https://abc.live.dynatrace...  Running      2d
```

---

# ⭐ **Section 7 — schema**

This is the **core**:
It defines what fields are allowed in a `DynaKube` object.

### Example:

```yaml
spec:
  activeGate:
    properties:
      capabilities:
        type: array
```

This defines the allowed fields in:

```yaml
spec:
  activeGate:
    capabilities:
      - routing
      - kubernetes-monitoring
      - metrics-ingest
```

The schema includes **hundreds** of fields, such as:

---

## 🔵 **activeGate section**

Fields for configuring ActiveGate pods:

* annotations
* capabilities
* customProperties
* env
* nodeSelector
* priorityClassName
* replicas
* resources
* tolerations
* topologySpreadConstraints

This defines *how* Dynatrace can deploy and control ActiveGate.

---

## 🔵 **oneAgent section (not fully shown in the snippet but present in full file)**

Defines:

* cloudNativeFullStack
* applicationMonitoring
* hostMonitoring
* classicFullStack

Plus tolerations, nodeSelectors, resources, etc.

---

## 🔵 **metadataEnrichment**

Whether to inject Dynatrace metadata into pods.

---

## 🔵 **metrics**

Defines:

* OpenMetrics scraping rules
* Prometheus annotations support

Example from CRD:

```yaml
spec.metrics.openMetrics
```

---

## 🔵 **logMonitoring**

Controls log ingestion settings.

---

# ⭐ **What CRDs Do NOT Contain**

CRDs **DO NOT** deploy anything.

They only define:

* valid fields
* allowed values
* validation rules
* what the `DynaKube` resource should look like

The **Dynatrace Operator** is the controller that “makes it happen.”

---

# ⭐ **Summary (Easy to Remember)**

### CRD = blueprint for a new Kubernetes API

### DynaKube CRD = blueprint for Dynatrace monitoring configuration

### Dynatrace Operator reads DynaKube CR and deploys:

✔ OneAgent
✔ ActiveGate
✔ CSI Driver
✔ Webhook
✔ Prometheus Scraping

This CRD describes the **entire configuration model** for Dynatrace behavior inside Kubernetes.

---



---

