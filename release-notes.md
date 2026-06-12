### 4. Third-Party Components and Controllers

### Karpenter Compatibility matrix
### present karpenter version 	karpenter-1.1.1 (helm)                      	1.1.1(API version)  

Kubernetes	1.30	1.31	1.32	1.33	1.34	1.35	1.36
karpenter	>= 0.37	>= 1.0.5	>= 1.2	>= 1.5	>= 1.6	>= 1.9	1.13.x

# 1. Check available versions in the OCI registry
helm search repo oci://public.ecr.aws/karpenter/karpenter --versions

# 2. Upgrade to v1.2.1 or higher (v1.2.1 is safe)
helm upgrade karpenter oci://public.ecr.aws/karpenter/karpenter/karpenter \
  --namespace kube-system \
  --version 1.2.1 \
  --reuse-values  # This preserves your existing configuration

# 3. Verify the upgrade
kubectl get pods -n kube-system | grep karpenter
kubectl get deployment karpenter -n kube-system -o jsonpath='{.spec.template.spec.containers[0].image}'


