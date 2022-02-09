#  Background on Admission Controllers
In the early days before Kubernetes became kubernetes (<v1.17, I think....), there were two driving design paradigms.  One was the scheduling algorithm and the other was job / workload intake.  The job/workload intake became known as the Admission Controller.  

# Admission Controllers
After one authenticates (you turned off anonymous authn/z, correct?), there is this middleware in Kubernete's stack that intercepts API requests.  The controller middleware (Admission) then reviews the request to perform some action.  Be it pubsubhub, webhook, mutate / modify, CRUD, etc...  Fast forward to today, there exists 2.5 kinds of Admission Controllers.  One is to validate the request.  Another is to mutate / manipulate the request, and the .5 is to perform validation & mutation.  For any reason the controller is configured to reject the request, the ENTIRE request in its entirety is rejected and, if required, operations rolled back (odd corner case for multi-clusters.)  Succinctly, it is a gatekeeper that sits behind the bouncer to determine who is allowed to enter the club but has the rights to kick out entire parties in case someone trying to get in doesn't fit the club's aesthetics.

#  Enablement
Typically, there are performed via plugins.  APIServer has a feature called enable-admission-plugins that accepts the controller plugins as parameters.  This feature will be invoked prior to orchestrating, queueing, and/or modifying objects in a cluster.  Within the APIServer's Admission Controller, there exists, currently, two distinct webhooks; MutatingAdmissionWebhook and ValidatingAdmissionWebHook.  They sent requests to external HTTP API endpoints and handle the response.  How they handle the response (and mutating the request-ish) is how they differ.  Validation webhooks can reject requests but not modify the response.  While mutating hooks may modify objects by patching the response.  Both return error responses if an error in incurred.

# Security controls love webhooks!
Need to enforce the lifecycle of workloads?  Need to provide assurance containers are not configured to run as root?  Need assurance.....  This is what makes APIServer's Admission Controllers great.  They are that gatekeeper that enables certain assurance statements to be made at the time the workload was instantiated and loaded into the cluster(s.)  This is where one will see a low of the low hanging security and compliance control fuits-of-labor wins come from this area of the cluster.  It doesn't scale with time nor with long running workloads that are not refreshed continiously.  It is only a point in time control, not time of use control (TOCTOU for only short-lived workloads.)


# Simple Example 
A common control for Production workloads is to perform vulnerability detection of the workload/job 's os packages' vulnerabilities.  This check is performed as a defense in depth as well as a compensating control in case the image repository's vuln. detection and mgt. feature does not operate as expected or if the CI vuln. detection controls fail.  Where the controls may be technical in nature or process.  AdmissionControllerImageVuln_BonzaiAnchore.yml contains a sample policy, helm chart, etc... to enable this functionality.

Here is a simple call out to Bonzai's Anchore to respond with its' findings.  If Allow all and warn bundle is enabled - all images are allowed, yet findings are emitted as warnings.  Reject critical bundle will reject workloads/job admission requests for images known to contain criticial (CVSS 8+) vulnerabilities.  Reject high bundle is same as critical, but for a lower CVSS score.  Or perhaps you wish to use the Reject root bundle that will respond and reject any workloads/jobs that attempt to use root.  Or maybe you want to use this tool as a hammer to reject ALL workloads/job requests by using the Deny All Images bundle.

One may test the failure by using the example below;
```
kubectl run --image=busybox -- sleep 3600
```
Which results in: Image failed policy check

Which one views the details on the policy failure, not much is said.   This is because the default bundle installed is Deny All Images bundle.

```
kubectl describe audits busybox1
Name:         busybox1
Namespace:    
Labels:       fakerelease=true
Annotations:  <none>
API Version:  security.banzaicloud.com/v1alpha1
Kind:         Audit
Metadata:
  Creation Timestamp:  2020-11-29T10:20:41Z
  Generation:          1
  Managed Fields:
    API Version:  security.banzaicloud.com/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .:
          f:fakerelease:
      f:spec:
        .:
        f:action:
        f:image:
        f:releaseName:
        f:resource:
        f:result:
      f:status:
        .:
        f:state:
    Manager:         anchore-image-validator
    Operation:       Update
    Time:            2020-11-29T10:20:41Z
  Resource Version:  39174
  Self Link:         /apis/security.banzaicloud.com/v1alpha1/audits/busybox1
  UID:               1e90c8b0-fffa-45f6-a986-d9fd269f0a83
Spec:
  Action:  reject
  Image:
    Image Digest:  
    Image Name:    
    Image Tag:     
    Last Updated:  
  Release Name:    busybox1
  Resource:        Pod
  Result:
    Image failed policy check: busybox
Status:
  State:  
Events:   <none>
```

As an exercise to the reader - look at the workload.  One will observe Bonzai needs root privileges.  For reasons that are not clear.  Which is why one will want to install the fundamental basic kubernete's policies and controls prior to running any production configuration workloads on the cluster(s.)  

Want to allow all images?  Apply AdmissionControllerAllowAllImages_BonzaiAnchore.yml .  Then one will be able to deploy the image failed above like this
```
kubectl run busybox1 --image=busybox -- sleep 3600
pod/busybox1 created
```

```
kubectl get whitelistitems -o wide -o=custom-columns=NAME:.metadata.name,CREATOR:.spec.creator,REASON:.spec.reason
NAME       CREATOR          REASON
busybox1   sephiroth    testing

kubectl get audits -o wide -o=custom-columns=NAME:.metadata.name,RELEASE:.spec.releaseName,IMAGES:.spec.image,RESULT:.spec.result
NAME       RELEASE    IMAGES                                                  RESULT
busybox1   busybox1   [map[imageDigest: imageName: imageTag: lastUpdated:]]   [Image failed policy check: busybox]
```

## Native Anchore
Here is a sample config for native Ancore.
```
apiVersion: v1
kind: Namespace
metadata:
  name: securty-system

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: anchore-engine
  namespace: kube-system
spec:
  repo: "https://charts.anchore.io"
  chart: anchore-engine
  targetNamespace: securty-system
  valuesContent: |-
    postgresql:
      image: centos/postgresql-96-centos7
      extraEnv:
      - name: POSTGRESQL_USER
        value: {anchore}
      - name: POSTGRESQL_PASSWORD
        value: {anchoresecret}
      - name: POSTGRESQL_DATABASE
        value: {anchoring}
      - name: PGUSER
        value: {postgresql}
      postgresPassword: {postgresotto}
      persistence:
        size: 10Gi
    anchoreGlobal:
      defaultAdminPassword: {anchoresecretapp}
      defaultAdminEmail: audit@haxx.ninja  
```


Here is a complex policy for the controller to enable certain types of image vuln. acceptance while rejecting blatantly negative vulnerabilities.
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: anchore-policys
data:
  production_bundle.json: |-
    {
        "blacklisted_images": [], 
        "comment": "Production bundle", 
        "id": "production_bundle", 
        "mappings": [
            {
                "id": "c4f9bf74-dc38-4ddf-b5cf-00e9c0074611", 
                "image": {
                    "type": "tag", 
                    "value": "*"
                }, 
                "name": "default", 
                "policy_id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "registry": "*", 
                "repository": "*", 
                "whitelist_ids": [
                    "37fd763e-1765-11e8-add4-3b16c029ac5c"
                ]
            }
        ], 
        "name": "production bundle", 
        "policies": [
            {
                "comment": "System default policy", 
                "id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "name": "DefaultPolicy", 
                "rules": [
                    {
                        "action": "STOP", 
                        "gate": "dockerfile", 
                        "id": "312d9e41-1c05-4e2f-ad89-b7d34b0855bb", 
                        "params": [
                            {
                                "name": "instruction", 
                                "value": "HEALTHCHECK"
                            }, 
                            {
                                "name": "check", 
                                "value": "not_exists"
                            }
                        ], 
                        "trigger": "instruction"
                    }, 
                    {
                        "action": "STOP", 
                        "gate": "vulnerabilities", 
                        "id": "b30e8abc-444f-45b1-8a37-55be1b8c8bb5", 
                        "params": [
                            {
                                "name": "package_type", 
                                "value": "all"
                            }, 
                            {
                                "name": "severity_comparison", 
                                "value": ">="
                            }, 
                            {
                                "name": "severity", 
                                "value": "high"
                            }
                        ], 
                        "trigger": "package"
                    }
                ], 
                "version": "1_0"
            }
        ], 
        "version": "1_0", 
        "whitelisted_images": [], 
        "whitelists": [
            {
                "comment": "Default global whitelist", 
                "id": "37fd763e-1765-11e8-add4-3b16c029ac5c", 
                "items": [], 
                "name": "Global Whitelist", 
                "version": "1_0"
            }
        ]
    }    
  testing_bundle.json: |-
    {
        "blacklisted_images": [], 
        "comment": "testing bundle", 
        "id": "testing_bundle", 
        "mappings": [
            {
                "id": "c4f9bf74-dc38-4ddf-b5cf-00e9c0074611", 
                "image": {
                    "type": "tag", 
                    "value": "*"
                }, 
                "name": "default", 
                "policy_id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "registry": "*", 
                "repository": "*", 
                "whitelist_ids": [
                    "37fd763e-1765-11e8-add4-3b16c029ac5c"
                ]
            }
        ], 
        "name": "Testing bundle", 
        "policies": [
            {
                "comment": "System default policy", 
                "id": "48e6f7d6-1765-11e8-b5f9-8b6f228548b6", 
                "name": "DefaultPolicy", 
                "rules": [
                    {
                        "action": "WARN", 
                        "gate": "dockerfile", 
                        "id": "312d9e41-1c05-4e2f-ad89-b7d34b0855bb", 
                        "params": [
                            {
                                "name": "instruction", 
                                "value": "HEALTHCHECK"
                            }, 
                            {
                                "name": "check", 
                                "value": "not_exists"
                            }
                        ], 
                        "trigger": "instruction"
                    }, 
                    {
                        "action": "STOP", 
                        "gate": "vulnerabilities", 
                        "id": "b30e8abc-444f-45b1-8a37-55be1b8c8bb5", 
                        "params": [
                            {
                                "name": "package_type", 
                                "value": "all"
                            }, 
                            {
                                "name": "severity_comparison", 
                                "value": ">"
                            }, 
                            {
                                "name": "severity", 
                                "value": "high"
                            }
                        ], 
                        "trigger": "package"
                    }
                ], 
                "version": "1_0"
            }
        ], 
        "version": "1_0", 
        "whitelisted_images": [], 
        "whitelists": [
            {
                "comment": "Default global whitelist", 
                "id": "37fd763e-1765-11e8-add4-3b16c029ac5c", 
                "items": [], 
                "name": "Global Whitelist", 
                "version": "1_0"
            }
        ]
    }    
  allow-all.json: |-
    {
      "blacklisted_images": [],
      "comment": "Allow all images and warn if vulnerabilities are found",
      "id": "allow_all_and_warn",
      "mappings": [
          {
              "id": "5fec9738-59e3-4c4c-9e74-281cbbe0337e",
              "image": {
                  "type": "tag",
                  "value": "*"
              },
              "name": "allow_all",
              "policy_id": "6472311c-e343-4d7f-9949-c258e3a5191e",
              "registry": "*",
              "repository": "*",
              "whitelist_ids": []
          }
      ],
      "name": "Allow all and warn bundle",
      "policies": [
          {
              "comment": "Allow all policy",
              "id": "6472311c-e343-4d7f-9949-c258e3a5191e",
              "name": "AllowAll",
              "rules": [
                  {
                      "action": "WARN",
                      "gate": "dockerfile",
                      "id": "bf8922ba-1f4e-4c4b-9057-165aa5f84b31",
                      "params": [
                          {
                              "name": "ports",
                              "value": "22"
                          },
                          {
                              "name": "type",
                              "value": "blacklist"
                          }
                      ],
                      "trigger": "exposed_ports"
                  },
                  {
                      "action": "WARN",
                      "gate": "dockerfile",
                      "id": "c44c6e6d-6d3f-4f20-971f-f5283b840e8f",
                      "params": [
                          {
                              "name": "instruction",
                              "value": "HEALTHCHECK"
                          },
                          {
                              "name": "check",
                              "value": "not_exists"
                          }
                      ],
                      "trigger": "instruction"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "6e04f5d8-27f7-47b9-b30a-de98fdf83d85",
                      "params": [
                          {
                              "name": "max_days_since_sync",
                              "value": "2"
                          }
                      ],
                      "trigger": "stale_feed_data"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "8494170c-5c3e-4a59-830b-367f2a8e1633",
                      "params": [],
                      "trigger": "vulnerability_data_unavailable"
                  },
                  {
                      "action": "WARN",
                      "gate": "vulnerabilities",
                      "id": "f3a89c1c-2363-4b6f-a05d-e784496ddb6f",
                      "params": [
                          {
                              "name": "package_type",
                              "value": "all"
                          },
                          {
                              "name": "severity_comparison",
                              "value": ">"
                          },
                          {
                              "name": "severity",
                              "value": "medium"
                          }
                      ],
                      "trigger": "package"
                  }
              ],
              "version": "1_0"
          }
      ],
      "version": "1_0",
      "whitelisted_images": [],
      "whitelists": []
    }    
---
apiVersion: batch/v1
kind: Job
metadata:
  name: anchore-policy-uplodaer
spec:
  template:
    metadata:
      name: anchore-policy-uplodaer
    spec:
      volumes:
      - name: anchore-policys
        configMap:
          name: anchore-policys
      containers:
      - name: anchore-policys
        image: "anchore/engine-cli"
        volumeMounts:
        - name: anchore-policys
          mountPath: /policy
        env:
        - name: ANCHORE_CLI_USER
          value: {anchorotto}
        - name: ANCHORE_CLI_PASS
          value: {anchoreclisecret}
        - name: ANCHORE_CLI_URL
          value: https://anchore-engine-anchore-engine-api:8228
        securityContext:
          runAsUser: 2
        command:
        - "sh"
        - "-c"
        - |
          set -ex
          anchore-cli policy add /policy/production_bundle.json
          anchore-cli policy add /policy/testing_bundle.json
          anchore-cli policy add /policy/allow-all.json          
      restartPolicy: OnFailure
```

In case you still haven't figured out how to remove Anchroe's root privileges, here is a pod policy to allow the suboptimal behavior

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: psp-rolebinding-securty-system
  namespace: securty-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system-unrestricted-psp-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:serviceaccounts

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: anchore-policy-validator
  namespace: kube-system
spec:
  repo: "https://charts.anchore.io/stable"
  chart: anchore-admission-controller
  targetNamespace: securty-system
  valuesContent: |-
    existingCredentialsSecret: anchore-credentials   
    anchoreEndpoint: "https://anchore-engine-anchore-engine-api:8228"
    policySelectors:
    - Selector:
        ResourceType: "pod"
        SelectorKeyRegex: "^breakglass$"
        SelectorValueRegex: "^true$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "testing_bundle"
      Mode: breakglass
    - Selector:
        ResourceType: "namespace"
        SelectorKeyRegex: "name"
        SelectorValueRegex: "^testing$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "testing_bundle"
      Mode: policy
    - Selector:
        ResourceType: "namespace"
        SelectorKeyRegex: "name"
        SelectorValueRegex: "^production$"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "production_bundle"
      Mode: policy
    - Selector:
        ResourceType: "image"
        SelectorKeyRegex: ".*"
        SelectorValueRegex: ".*"
      PolicyReference:
        Username: "admin"
        PolicyBundleId: "allow-all"
      Mode: breakglass  
```


Always sanity check and build these steps into your monitoring and deployment pipelines

```
kubectl run -i -t anchorecli --image anchore/engine-cli --restart=Always \
--env ANCHORE_CLI_URL=https://anchore-engine-anchore-engine-api:8228 \
--env ANCHORE_CLI_USER=${anchore} \
--env ANCHORE_CLI_PASS=${anchorsecret}

anchore-cli policy list

anchore-cli image add nginx
anchore-cli image list

anchore-cli evaluate check alpine --policy testing_bundle
anchore-cli evaluate check alpine --policy production_bundle
```

Create the testing, production, and www namespaces.  Then verify the policies and controllers fail you.

```
kubectl -n testing run -it alpine --restart=Never --image alpine /bin/sh                                                                               
If you don't see a command prompt, try pressing enter.
/ #


kubectl -n production run -it alpine --restart=Never --image alpine /bin/sh
Error from server: admission webhook "anchore-admission-controller-admission.anchore.io" denied the request: Image alpine with digest sha256:e103c1b4bf019dc290bcc7aca538dc2bf7a9d0fc836e186f5fa34945c5168310 failed policy checks for policy bundle production_bundle

kubectl -n production run -it alpine --labels="breakglass=true" --restart=Never --image alpine /bin/sh
If you don't see a command prompt, try pressing enter.
```

Add breakglass=true to bypass the checks (see policy above to enable breakglass parameter.)
```
kubectl -n production run -it alpine --restart=Never --labels="breakglass=true" --image alpine /bin/sh
If you don't see a command prompt, try pressing enter.
/ # exit 
```
