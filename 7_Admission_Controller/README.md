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
