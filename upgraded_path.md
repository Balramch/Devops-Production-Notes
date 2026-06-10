### cluster version 1.31 to 1.32
##### 1. Upgrade addons

```
aws eks update-addon \
  --cluster-name genesys-prod-cluster \
  --addon-name coredns \
  --addon-version v1.11.4-eksbuild.33
```


```
aws eks update-addon \
  --cluster-name genesys-prod-cluster \
  --addon-name kube-proxy \
  --addon-version v1.32.13-eksbuild.5
```

```
aws eks update-addon \
  --cluster-name genesys-prod-cluster \
  --addon-name vpc-cni \
  --addon-version v1.20.5-eksbuild.1
```

```
aws eks update-addon \
  --cluster-name genesys-prod-cluster \
  --addon-name eks-pod-identity-agent \
  --addon-version v1.3.10-eksbuild.3
```


#### 2. Upgrade Control Plane
```
aws eks update-cluster-version \
  --name genesys-prod-cluster \
  --kubernetes-version 1.32
```
### Wait until

```
aws eks describe-cluster \
  --name genesys-prod-cluster \
  --query cluster.status
```

#### 3. Upgrade Managed Node Group

```
aws eks update-nodegroup-version \
  --cluster-name genesys-prod-cluster \
  --nodegroup-name prod-managednodegroup-karpenter-v2 \
  --kubernetes-version 1.32
```


## Note : Befor upgrade check

```
kubectl get pdb -A
kubectl get nodes -o wide
aws eks list-addons --cluster-name genesys-prod-cluster
```


### another method

### Step 3: The Upgrade Sequence of Operations

Regardless of your chosen method, the physical sequence of upgrades is critical. **The order below is strict and must be followed.**

1. **Perform Pre-Checks**: Verify the Kubernetes API compatibility using a tool like `kubent` or `pluto` and double-check your `PodDisruptionBudgets`.
    
2. **Upgrade Third-Party Controllers** (Before Control Plane): Upgrade controllers like `cluster-autoscaler` or `karpenter` to their versions compatible with **Kubernetes 1.32** while your cluster is still on version 1.31[](https://dev.to/tallgray1/stop-managing-eks-add-ons-by-hand-1cc4).
    
3. **Upgrade the EKS Control Plane**: Use the AWS Console, CLI, or `eksctl` to upgrade the cluster from 1.31 to 1.32. This can take several minutes[](https://repost.aws/ja/knowledge-center/eks-upgrade-cluster-add-custom-controllers?sc_ichannel=ha&sc_ilang=ja&sc_isite=repost&sc_iplace=hp&sc_icontent=AAwUmmKjoqQF-P2VkEO-jkRQ&sc_ipos=11)[](https://docs.aws.amazon.com/ja_jp/eks/latest/eksctl/AmazonEksEksctlDocs.pdf#21#9).
    
4. **Upgrade EKS Add-ons (CoreDNS, kube-proxy, VPC CNI)**: After the control plane is upgraded, update your core add-ons to their 1.32-compatible versions. If using `eksctl`:
    
    bash
    
    # Example for updating coredns
    eksctl utils update-coredns --cluster=<your-cluster-name> --approve
    
    _If you migrate them to be EKS-managed, you can update them directly via the AWS Console or `aws eks update-addon` commands[](https://repost.aws/ja/knowledge-center/eks-upgrade-cluster-add-custom-controllers?sc_ichannel=ha&sc_ilang=ja&sc_isite=repost&sc_iplace=hp&sc_icontent=AAwUmmKjoqQF-P2VkEO-jkRQ&sc_ipos=11)[](https://repost.aws/knowledge-center/eks-upgrade-cluster-add-custom-controllers?sc_ichannel=ha&sc_ilang=fr&sc_isite=repost&sc_iplace=hp&sc_icontent=AAKEJxajEJReGc2bj7dSyojQ&sc_ipos=12)._
    
5. **Upgrade Worker Nodes**: The control plane upgrade does not touch your nodes. You must either update the launch template for self-managed nodes or use `eksctl upgrade nodegroup` for managed node groups. The goal is to cycle all nodes to a version that matches your new control plane[](https://www.linkedin.com/posts/vishal-yadav-b8b794278_aws-eks-kubernetes-activity-7391711162507120640-4Z3S).
    
6. **Post-Upgrade Validation**: Run `kubectl get nodes` and `kubectl get pods -A` to verify everything is back to a `Ready` and `Running` state.
    

Would you like more detailed, step-by-step commands for migrating your cluster's core add-ons to the fully managed AWS model before you begin the version upgrade?