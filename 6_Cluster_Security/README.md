#  Cluster policies and governance
For Production kubernetes' clusters, it is important to adhere to separation of privileges as well as least privileges for operators.   As a result, it is very important to enforce cluster-wide policies to govern what containers and similar workloads are permitted to perform.  Pod Security goes some of the way.  But this is deeper within the stack - at the cluster level.

# OPA
Open Policy Agent (OPA), is a policy engine for Cloud Native environments hosted by CNCF. It is a general purpose policy engine. OPA policies are written in a Domain Specific Language (DSL) called Rego.

# OPA Gatekeeper
Gatekeeper is specifically built for Kubernetes Admission Control's OPA side car use cases.  The stack utilizes OPA internally, but specifically instrumenting the admission controller. Compared to using OPA with its sidecar kube-mgmt (aka Gatekeeper v1.0) techniques, >v1.0 Gatekeeper is integrated with the OPA Constraint Framework to enforce CRUD-based policies and allow declaratively configured policies to be reliably shared with repeatable, predictable outcomes across clusters.  Simple install via
```
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/master/deploy/gatekeeper.yaml
```

# Supply Chain Risk management
A simple example of this amazing tool and technique is to manage third party risks.  There are risky image repositories or potential controls implemented where only certain registries are allowed to be used.  The thinking unauthorized or sketchy registries may contain "the official RHEL container image" yet actually be backdoored and will happily run a monero crypto miner in the background.  Below is an example in Rego about only allowing gcr.io images
```
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredregistry
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredRegistry
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            image:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredregistry
        violation[{"msg": msg, "details": {"Registry should be": required}}] {
          input.review.object.kind == "Pod"
          some i
          image := input.review.object.spec.containers[i].image
          required := input.parameters.registry
          not startswith(image,required)
          msg := sprintf("Forbidden registry: %v", [image])
        } 
```

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredRegistry
metadata:
  name: images-must-come-from-gcr
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    registry: "gcr.io/"
```

Validate the control's effectiveness by failing to run an image from github's repository.
```
kubectl run --generator=run-pod/v1 busybox1 --image=busybox -- sleep 3600
```

# OPA Gatekeeper auditability
OPA Gatekeeper has a wonderful (limited) audit functionality built it.  Violations are stored in the status field of the failed constraint.  Querying the failures is as simple as;
```
kubectl describe policystrictonly.constraints.gatekeeper.sh policy-strict-constraint
```


# Kyverno
OPA Gatekeeper is not the only policy engine.  With Kyverno, policies are managed as Kubernetes resources and no new language is required to write policies. This allows using familiar tools such as kubectl, git, and kustomize to manage policies. Kyverno policies can validate, mutate, and generate Kubernetes resources. The Kyverno CLI can be used to test policies and validate resources as part of a CI/CD pipeline.   Here is the same supply chain risk mgt. policy control listed as above but written for Kyverno
```
apiVersion : kyverno.io/v1alpha1
kind: Policy
metadata:
  name: check-registries
spec:
  rules:
  - name: check-registries
    resource:
      kinds:
      - Deployment
      - StatefulSet
    validate:
      message: "Registry is not allowed"
      pattern:
        spec:
          template:
            spec:
              containers:
              - name: "*"
                # Check allowed registries
                image: "*/gcr.io/*"
```

It is important to note that this policy only applies to resources within the namespace the policy defines, not at the cluster layer.

# Mutations
Or maybe you wish to modify a policy without CRUD'ing it directly.  Here is simple json patch to add a label
```
apiVersion : kyverno.io/v1alpha1
kind : Policy
metadata :
  name : policy-deployment
spec :
  rules:
    - name: patch-add-label
      resource:
        kinds : 
        - Deployment
      mutate:
        patches:
        - path: /metadata/labels/isMutated
          op: add
          value: "true"
```

Or maybe you wish to have situational or conditional logic statements

```
apiVersion: kyverno.io/v1alpha1
kind: Policy
metadata:
  name: set-image-pull-policy
spec:
  rules:
  - name: set-image-pull-policy
    resource:
      kinds:
      - Deployment
    mutate:
      overlay:
        spec:
          template:
            spec:
              containers:
                # if the image tag is latest, set the imagePullPolicy to Always
                - (image): "*:latest"
                  imagePullPolicy: "Always"
```

# Kyverno Reporting
Kyverno reporting provides detailed information about execution and violations.  They are stored per namespace and cluster.  A namespace query
```
kubectl get polr -A
```

or cluster layer
```
kubectl get clusterpolicyreport -A
```

with describing the events in more details such as;
```
kubectl describe polr polr-ns-default | grep "Status: \+fail" -B10
  Message:        validation error: Running as root is not allowed. The fields spec.securityContext.runAsNonRoot, spec.containers[*].securityContext.runAsNonRoot, and spec.initContainers[*].securityContext.runAsNonRoot must be `true`. Rule check-containers[0] failed at path /spec/securityContext/runAsNonRoot/. Rule check-containers[1] failed at path /spec/containers/0/securityContext/.
  Policy:         require-run-as-non-root
  Resources:
    API Version:  v1
    Kind:         Pod
    Name:         add-capabilities-init-containers
    Namespace:    default
    UID:          1caec743-faed-4d5a-90f7-5f4630febd58
  Rule:           check-containers
  Scored:         true
  Status:         fail
--
  Message:        validation error: Running as root is not allowed. The fields spec.securityContext.runAsNonRoot, spec.containers[*].securityContext.runAsNonRoot, and spec.initContainers[*].securityContext.runAsNonRoot must be `true`. Rule check-containers[0] failed at path /spec/securityContext/runAsNonRoot/. Rule check-containers[1] failed at path /spec/containers/0/securityContext/.
  Policy:         require-run-as-non-root
  Resources:
    API Version:  v1
    Kind:         Pod
    Name:         sysctls
    Namespace:    default
    UID:          b98bdfb7-10e0-467f-a51c-ac8b75dc2e95
  Rule:           check-containers
  Scored:         true
  Status:         fail
```


## Cluster layer policy engines
There exists a large number of CNCF-blessed policy engines.  There exists N times more on github, bit bucket, and within private corporations.  Each has their pros and cons.  Find the one that works best within the constraints of the project and lifecycle.  I prefer OPA Gatekeeper.
