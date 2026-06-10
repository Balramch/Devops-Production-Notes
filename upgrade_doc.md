
### From cluster version 1.31 to 1.32 
### 1. Cluster Add-ons (CoreDNS, kube-proxy, VPC CNI)
```
# List current add-ons in your cluster
aws eks list-addons --cluster-name genesys-prod-cluster
```

#### Response

```
aws eks list-addons --cluster-name genesys-prod-cluster

{

    "addons": [

        "coredns",

        "eks-pod-identity-agent",

        "kube-proxy",

        "vpc-cni"

    ]

}
```


#### Verify if your add-on versions are compatible with 1.32
#### Replace addon-names with: coredns, kube-proxy, vpc-cni
```
aws eks describe-addon-versions --kubernetes-version 1.32 --addon-name <addon-name>
```


### Recommended Version of Addons for 1.32
### Easy way to check the compatible versions of kube-proxy
```
aws eks describe-addon-versions \
  --addon-name kube-proxy \
  --kubernetes-version 1.32 \
  --query 'addons[].addonVersions[?compatibilities[?defaultVersion==`true`]].addonVersion' \
  --output text
```
### Result 
```
v1.32.13-eksbuild.5
```

### For vpc-cni
```
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.32 \
  --query 'addons[].addonVersions[?compatibilities[?defaultVersion==`true`]].addonVersion' \
  --output text
```
## result
```
v1.20.5-eksbuild.1
```


###  For core-dns
```
aws eks describe-addon-versions \
  --addon-name coredns \
  --kubernetes-version 1.32 \
  --query 'addons[].addonVersions[?compatibilities[?defaultVersion==`true`]].addonVersion' \
  --output text
```


#### Result
```
v1.11.4-eksbuild.33
```

### For eks-pod-identity-agent

```
 aws eks describe-addon-versions \

  --addon-name eks-pod-identity-agent \

  --kubernetes-version 1.32 \                                   

  --query 'addons[].addonVersion' \
  -- output text
```
### Result
```
v1.3.10-eksbuild.3
```


### Current versions of addons on 1.31 cluster
### Current version of addon eks-pod-identity-agent
```
aws eks describe-addon \

  --cluster-name genesys-prod-cluster \

  --addon-name eks-pod-identity-agent \

 --query 'addon.addonVersion' \

  --output text
```

#### Result
```
v1.3.4-eksbuild.1
```

### Current version of addon for vpc-cni
```
 aws eks describe-addon \

  --cluster-name genesys-prod-cluster \

  --addon-name vpc-cni \               

  --query 'addon.addonVersion' \

  --output text
```

### Result
```
v1.19.0-eksbuild.1
```

### Current Version of addon for coredns
```
aws eks describe-addon \

  --cluster-name genesys-prod-cluster \

  --addon-name coredns \ 

  --query 'addon.addonVersion' \

  --output text
```

### Result
```
v1.11.3-eksbuild.1
```

#### Current version of addon for kube-proxy

```
aws eks describe-addon \

  --cluster-name genesys-prod-cluster \

  --addon-name kube-proxy \

  --query 'addon.addonVersion' \

  --output text
```

### Result
```
v1.31.2-eksbuild.3
```


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




### 2. Node Group AMI Compatibility

#### Check your node groups

```
aws eks list-nodegroups \

  --cluster-name genesys-prod-cluster

```

### result
```
{

    "nodegroups": [

        "prod-managednodegroup-karpenter-v2"

    ]
 }
```



### Check node group details

```
aws eks describe-nodegroup \
  --cluster-name genesys-prod-cluster \
  --nodegroup-name prod-managednodegroup-karpenter-v2
```

### Check the latest AMI release for Kubernetes 1.32
```
 aws ssm get-parameter \

  --name /aws/service/eks/optimized-ami/1.32/amazon-linux-2023/x86_64/standard/recommended/release_version
```
#### Response
```
{

    "Parameter": {

        "Name": "/aws/service/eks/optimized-ami/1.32/amazon-linux-2023/x86_64/standard/recommended/release_version",

        "Type": "String",

        "Value": "1.32.13-20260529",

        "Version": 61,

        "LastModifiedDate": "2026-06-06T03:00:29.421000+05:45",

        "ARN": "arn:aws:ssm:eu-central-1::parameter/aws/service/eks/optimized-ami/1.32/amazon-linux-2023/x86_64/standard/recommended/release_version",

        "DataType": "text"

    }

}
```

### For AL2
```
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.32/amazon-linux-2/recommended/release_version
```

## Response
```
{

    "Parameter": {

        "Name": "/aws/service/eks/optimized-ami/1.32/amazon-linux-2/recommended/release_version",

        "Type": "String",

        "Value": "1.32.9-20251209",

        "Version": 42,

        "LastModifiedDate": "2025-12-13T02:02:22.151000+05:45",

        "ARN": "arn:aws:ssm:eu-central-1::parameter/aws/service/eks/optimized-ami/1.32/amazon-linux-2/recommended/release_version",

        "DataType": "text"

    }

}
```


### Check if your node group can be upgraded to 1.32

```
aws eks update-nodegroup-version \
  --cluster-name genesys-prod-cluster \
  --nodegroup-name prod-managednodegroup-karpenter-v2 \
  --kubernetes-version 1.32 \
  --dry-run
```

### not work

#### Quick compatibility check

```
aws eks describe-nodegroup \

  --cluster-name genesys-prod-cluster \

  --nodegroup-name prod-managednodegroup-karpenter-v2 \

  --query 'nodegroup.{Version:version,AMI:releaseVersion,AMIType:amiType}'
```

### Response
```
{

    "Version": "1.31",

    "AMI": "1.31.5-20250212",

    "AMIType": "AL2_x86_64"

}
```


```

``` aws ssm get-parameter \

  --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2/recommended/release_version \

  --query Parameter.Value \

  --output text

  

aws ssm get-parameter \

  --name /aws/service/eks/optimized-ami/1.32/amazon-linux-2/recommended/release_version \

  --query Parameter.Value \

  --output text

1.31.13-20251209

1.32.9-20251209


#### 3. Remove kubernetes deprecated apis

#### Identify what is actually remove
```
kubectl get --raw /metrics | grep deprecated
```

### With the tool pluto

```
 pluto detect-api-resources -o wide

I0610 13:46:31.635839    5228 warnings.go:107] "Warning: v1 ComponentStatus is deprecated in v1.19+"
```


### 4. Third-Party Components and Controllers


### 5. PodDisruptionBudgets (PDBs) and Topology Spread Constraints
```
kubectl get pdb --all-namespaces
```

```
kubectl get pdb --all-namespaces

NAMESPACE      NAME                   MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE

istio-system   istio-ingressgateway   1               N/A               0                     283d

istio-system   istiod                 1               N/A               3                     283d

kube-system    coredns                N/A             1                 1                     511d

kube-system    karpenter              N/A             1                 1                     511d

kube-system    metrics-server         1               N/A               1                     506d

balram@Balrams-MacBook-Air ~ % kubectl describe pdb istio-ingressgateway -n istio-system

Name:           istio-ingressgateway

Namespace:      istio-system

Min available:  1

Selector:       app=istio-ingressgateway,istio=ingressgateway

Status:

    Allowed disruptions:  0

    Current:              1

    Desired:              1

    Total:                1

Events:                   <none>
```


### 6. Subnet IP Capacity

```
CLUSTER=genesys-prod-cluster
aws ec2 describe-subnets --subnet-ids \
  $(aws eks describe-cluster --name ${CLUSTER} \
  --query 'cluster.resourcesVpcConfig.subnetIds' \
  --output text) \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,AvailableIpAddressCount]' \
  --output table
```

### 7. Cluster Health and Insights
