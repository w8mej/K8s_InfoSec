# ImagePullSecrets
ImagePullSecrets is an odd feature to kubernetes.  It allows one to handle container registry secrets.  However, instead of cluster-wide or similar policy-driven behaviors, it is only accessible on a per pod or namespace nomenclature.   There are a few nifty hacks to enable a cluster to utilize a "global" container secret for acquiring and pulling workload images.

# Kyverno
Here is a simple policy to enable just that on a cluster-wide basis.  It will allow all images to be pulled from a container hosting provider with a given pre-defined secret called image-pull-secret.
```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-secret
spec:
  background: false
  rules:
  - name: sync-image-pull-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: Secret
      name: image-pull-secret
      namespace: "{{request.object.metadata.name}}"
      synchronize: true
      clone:
        namespace: default
        name: image-pull-secret
---
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutate-imagepullsecret
spec:
  rules:
    - name: mutate-imagepullsecret
      match:
        resources:
          kinds:
          - Pod
      mutate:
        patchStrategicMerge:
          spec:
            imagePullSecrets:
            - name: image-pull-secret  
            (containers):
            - (image): "*" 
```

# TitanSoft
TitanSoft open-sourced a simple Kubernetes application called imagepullsecret-patcher, which automatically creates and patches imagePullSecrets to default service accounts in all Kubernetes namespaces to allow cluster-wide authenticated access to private container registry.

## How to
First, setup your container registry credentials.  Below, we will use Docker.  GCR and ECR supports JSON key file authentication method (json_key username, service account private key as password.)
```
kubectl create secret docker-registry image-pull-secret \
  -n <your-namespace> \
  --docker-server=<your-registry-server> \
  --docker-username=<your-name> \
  --docker-password=<your-password> \
  --docker-email=<your-email>
```

### Approach 1 - Per Pod basis
Place it into each pod definition
```
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: my-namespace
spec:
  containers:
    - name: my-app
      image: my-private-registry/my-app:v1
  imagePullSecrets:
    - name: image-pull-secret
```

### Approach 2 - Default ServiceAccount or the applicable ServiceAccount
```
kubectl patch serviceaccount default \
  -p "{\"imagePullSecrets\": [{\"name\": \"image-pull-secret\"}]}" \
  -n <your-namespace>
```

 <a href="https://k8s.haxx.ninja"><img src="https://github.com/w8mej/K8s_InfoSec/raw/main/12_PullingImages/ImagePuller.png" alt="w8mej k8s image signing" width="600"></a>


