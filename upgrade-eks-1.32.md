# EKS Kubernetes 1.31 → 1.32 Upgrade Compatibility Guide

> **Note:** This document covers third-party component compatibility checks and upgrade recommendations for upgrading an Amazon EKS cluster from Kubernetes **1.31** to **1.32**.

---

# Third-Party Components and Controllers

## Karpenter Compatibility Matrix

### Current Version

| Component   | Version         |
| ----------- | --------------- |
| Helm Chart  | karpenter-1.1.1 |
| API Version | 1.1.1           |

### Kubernetes Compatibility

| Kubernetes | 1.30   | 1.31    | 1.32  | 1.33  | 1.34  | 1.35  | 1.36   |
| ---------- | ------ | ------- | ----- | ----- | ----- | ----- | ------ |
| Karpenter  | >=0.37 | >=1.0.5 | >=1.2 | >=1.5 | >=1.6 | >=1.9 | 1.13.x |

### Verify Existing Resources

```bash
# Check if you're using Provisioners
kubectl get provisioners -A

# Check for NodePools (new API)
kubectl get nodepools -A
```

### Check Available Versions

```bash
helm search repo oci://public.ecr.aws/karpenter/karpenter --versions
```

### Upgrade Karpenter

```bash
helm upgrade karpenter oci://public.ecr.aws/karpenter/karpenter/karpenter \
  --namespace kube-system \
  --version 1.2.1 \
  --reuse-values
```

### Verify Upgrade

```bash
kubectl get pods -n kube-system | grep karpenter

kubectl get deployment karpenter -n kube-system \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

# KEDA

Reference:
https://keda.sh/docs/2.20/operate/cluster/

## Kubernetes Compatibility

| KEDA Version | Kubernetes Version |
| ------------ | ------------------ |
| v2.20        | v1.33 - v1.35      |
| v2.19        | v1.32 - v1.34      |
| v2.18        | v1.31 - v1.33      |
| v2.17        | v1.30 - v1.32      |
| v2.16        | v1.29 - v1.31      |
| v2.15        | v1.28 - v1.30      |
| v2.14        | v1.27 - v1.29      |
| v2.13        | v1.27 - v1.29      |
| v2.12        | v1.26 - v1.28      |
| v2.11        | v1.25 - v1.27      |
| v2.10        | v1.24 - v1.26      |
| v2.9         | v1.23 - v1.25      |
| v2.8         | v1.17 - v1.25      |
| v2.7         | v1.17 - v1.25      |

### Current Version

* KEDA: **2.16.1**

### Recommended Upgrade

* Upgrade to **2.19.0** for Kubernetes 1.32 support.

```bash
helm upgrade keda kedacore/keda \
  --namespace keda \
  --version 2.19.0 \
  --set crds.install=true
```

---

# External Secrets Operator (ESO)

Reference:
https://external-secrets.io/latest/introduction/stability-support/

## Current Version

| Component   | Version                 |
| ----------- | ----------------------- |
| Helm Chart  | external-secrets-0.10.5 |
| API Version | v0.10.5                 |

## Compatibility Summary

| ESO Version | Kubernetes Version |
| ----------- | ------------------ |
| 0.16.x      | 1.32               |
| 0.15.x      | 1.32               |
| 0.14.x      | 1.32               |
| 0.13.x      | 1.19 → 1.31        |
| 0.12.x      | 1.19 → 1.31        |
| 0.11.x      | 1.19 → 1.31        |
| 0.10.x      | 1.19 → 1.31        |

### Recommended Upgrade

Upgrade from **0.10.5** to at least **0.16.x** before or during the Kubernetes 1.32 upgrade.

### Upgrade Steps

#### 1. Apply Updated CRDs

```bash
kubectl apply -f https://githubusercontent.com
```

#### 2. Update Helm Repository

```bash
helm repo update external-secrets
```

#### 3. Upgrade Helm Release

```bash
helm upgrade external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --version 0.16.0 \
  --reuse-values
```

#### 4. Verify Rollout

```bash
kubectl rollout status deployment external-secrets \
  --namespace external-secrets
```

---

# AWS Load Balancer Controller

## Current Version

* v2.10.0

## Compatibility

* Kubernetes 1.32 supports AWS Load Balancer Controller v2.10.0.
* Recommended version range:

  * v2.11.0+
  * Latest stable v2.x release preferred.

### Upgrade

```bash
helm repo update

helm upgrade aws-load-balancer-controller aws/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.11.0
```

---

# kube-prometheus-stack

## Current Environment

| Component           | Current Version |
| ------------------- | --------------- |
| Helm Chart          | v68.4.4         |
| Prometheus Operator | v0.79.2         |
| kube-state-metrics  | v2.13.0         |

## Recommended for Kubernetes 1.32

| Component           | Recommended Version | Reason                                               |
| ------------------- | ------------------- | ---------------------------------------------------- |
| Helm Chart          | v80.x.x - v86.x.x+  | Includes Kubernetes 1.32 API compatibility fixes     |
| Prometheus Operator | v0.84.0 - v0.91.0+  | Improved schema validation support                   |
| kube-state-metrics  | v2.15.x - v2.16.x   | Fixes missing metrics for newer Kubernetes resources |

### Backup Existing Values

```bash
helm get values kube-prometheus-stack -n monitoring \
  > kps-values-backup.yaml
```

### Upgrade CRDs

```bash
kubectl apply -f https://githubusercontent.com
```

### Upgrade Helm Chart

```bash
helm repo update

helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f kps-values-backup.yaml \
  --version 82.4.0
```

---

# Loki Stack

## Current Version

| Component  | Version           |
| ---------- | ----------------- |
| Helm Chart | loki-stack-2.10.2 |
| Loki API   | v2.9.3            |

## Kubernetes 1.32 Recommendation

* Verify Loki chart compatibility with Kubernetes 1.32.
* Recommended:

  * Loki >= 3.x chart versions
  * Loki >= 2.9.x runtime version

### Verify Current Installation

```bash
helm list -A | grep loki

kubectl get pods -A | grep loki
```

### Check Available Versions

```bash
helm repo update

helm search repo grafana/loki-stack --versions
```

### Upgrade Example

```bash
helm upgrade loki grafana/loki-stack \
  --namespace monitoring \
  --reuse-values
```
### Istio 
### Check istio version
```
 istioctl version
### response
client version: 1.30.1
control plane version: 1.25.0
data plane version: 1.25.0 (1 proxies)
```
```
 kubectl -n istio-system get deployment istiod \
-o jsonpath='{.spec.template.spec.containers[0].image}'
### Response
docker.io/istio/pilot:1.25.0  
```
### precheck for next version
```
 istioctl x precheck
```

### Support status link for istio release
https://istio.io/latest/docs/releases/supported-releases/
### In our case 1.32 supports this istio but recommend to upgrade other istion version 1.26 

### Argocd Compatibility URL
https://argo-cd.readthedocs.io/en/release-3.4/operator-manual/tested-kubernetes-versions/
#### Check the current Argocd version
```
 kubectl -n argocd get pod -l app.kubernetes.io/name=argocd-server -o jsonpath='{.items[0].spec.containers[0].image}'
```
```
 kubectl -n argocd get pods -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.containers[0].image}{"\n"}{end}'
```
### Response
argocd:v3.4.2



---

# Pre-Upgrade Validation Checklist

* [ ] Verify deprecated APIs are not in use.
* [ ] Verify Karpenter compatibility.
* [ ] Upgrade KEDA to 2.19.0.
* [ ] Upgrade External Secrets Operator to 0.16.x or later.
* [ ] Upgrade AWS Load Balancer Controller to latest supported release.
* [ ] Upgrade kube-prometheus-stack to 80.x+.
* [ ] Verify Loki compatibility.
* [ ] Upgrade EKS managed add-ons.
* [ ] Upgrade worker node AMIs.
* [ ] Validate application workloads after upgrade.

---

# Post-Upgrade Verification

```bash
kubectl get nodes

kubectl get pods -A

kubectl get events -A --sort-by=.lastTimestamp

kubectl api-resources

kubectl top nodes

kubectl top pods -A
```

