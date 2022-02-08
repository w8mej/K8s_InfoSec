# Pod Security - Why?!
Pod Security is the successor to PodSecurityPolicy (PSP - EOL 1.25.) The intent behind Pod Security is to mitigate controller deployment challenges (OPA as a compensating control) and lacking dry operations's functionality (dry-run, audit, etc...


## PSP or Pod Security?
Pod Security use profiles based on Pod Security Standards - https://kubernetes.io/docs/concepts/security/pod-security-standards/ . The three policy levels defined are as below:

privileged - The Privileged policy is purposely-open, and entirely unrestricted.

baseline - The Baseline policy is aimed at ease of adoption for common containerized workloads while preventing known privilege escalations.
Disables HostProcess for windows
Disables Host Namespaces on linux like: hostNetwork, hostPID and hostIPC
Disables Privileged Containers
Disallow adding of Capabilities
Disallow mounting of HostPath Volumes
Disallow usage of Host Ports
On supported hosts, the runtime/default AppArmor profile is applied by default.
Setting the SELinux type is restricted, and setting a custom SELinux user or role option is forbidden.
Seccomp profile must not be explicitly set to Unconfined.
Disallow the configuration of Sysctls

restricted - The Restricted policy is aimed at enforcing current Pod hardening best practices.
Disallow Privilege Escalation
Disallow running cotainer as root user, group and uid or guid as 0
Seccomp profile must be explicitly set to one of the allowed values.
Containers must drop ALL capabilities, and are only permitted to add back the NET_BIND_SERVICE capability.

## Applying Pod Security policies
Policies are applied in a specific mode. The modes are:

enforce — Any Pods that violate the policy will be rejected
audit — Violations will be recorded as an annotation in the audit logs, but don’t affect whether the pod is allowed.
warn — Violations will send a warning message back to the user, but don’t affect whether the pod is allowed.)


### Example namespace policy deployed
```kubectl create ns verify-pod-security
kubens verify-pod-security

#### enforces a "restricted" security policy and audits on restricted
kubectl label --overwrite ns verify-pod-security \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted
```

### Atttempt to fail deploying a privileged (restricted above) compute workload
cat <<EOF | kubectl -n verify-pod-security apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-privileged
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
    securityContext:
      allowPrivilegeEscalation: true
EOF


Observe the failed response.  Now try with the accepted privileges on the compute workload
```kubectl label --overwrite ns verify-pod-security \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/audit=baseline
```


## Another lever to apply policies - clusterwide
Instead of deploying just to the namespace, one is able to apply pod security policies clusterwide.
```kind create cluster --image kindest/node:v1.23.0 --config kind-config.yaml && kubectl cluster-info --context kind-kind && kubectx kind-kind

### Create a test namespace and attempt to deploy a highly privileged compute workload.  Observe the failure.
kubectl create namespace test-defaults
namespace/test-defaults created

$ kubectl describe namespace test-defaults
Name:         test-defaults
Labels:       kubernetes.io/metadata.name=test-defaults
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.

cat <<EOF | kubectl -n test-defaults apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: busybox-privileged
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - sleep
    - "1000000"
    securityContext:
      allowPrivilegeEscalation: true
EOF

Observe the metrics reporting endpoint measuring the failures
kubectl get --raw /metrics | grep pod_security_evaluations_total

# HELP pod_security_evaluations_total [ALPHA] Number of policy evaluations that occurred, not counting ignored or exempt requests.
# TYPE pod_security_evaluations_total counter
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 2
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="privileged",policy_version="latest",request_operation="create",resource="pod",subresource=""} 0
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="privileged",policy_version="latest",request_operation="update",resource="pod",subresource=""} 0
pod_security_evaluations_total{decision="deny",mode="audit",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 1
pod_security_evaluations_total{decision="deny",mode="enforce",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 1
pod_security_evaluations_total{decision="deny",mode="warn",policy_level="restricted",policy_version="latest",request_operation="create",resource="controller",subresource=""} 2
pod_security_evaluations_total{decision="deny",mode="warn",policy_level="restricted",policy_version="latest",request_operation="create",resource="pod",subresource=""} 2



# Metrics audit policy
k8s-metrics-audit.yml contains a few pod security events one may wish to observe.  
