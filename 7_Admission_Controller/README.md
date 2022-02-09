#  Background on Admission Controllers
In the early days before Kubernetes became kubernetes (<v1.17, I think....), there were two driving design paradigms.  One was the scheduling algorithm and the other was job / workload intake.  The job/workload intake became known as the Admission Controller.  

# Admission Controllers
After one authenticates (you turned off anonymous authn/z, correct?), there is this middleware in Kubernete's stack that intercepts API requests.  The controller middleware (Admission) then reviews the request to perform some action.  Be it pubsubhub, webhook, mutate / modify, CRUD, etc...  Fast forward to today, there exists 2.5 kinds of Admission Controllers.  One is to validate the request.  Another is to mutate / manipulate the request, and the .5 is to perform validation & mutation.  For any reason the controller is configured to reject the request, the ENTIRE request in its entirety is rejected and, if required, operations rolled back (odd corner case for multi-clusters.)  Succinctly, it is a gatekeeper that sits behind the bouncer to determine who is allowed to enter the club but has the rights to kick out entire parties in case someone trying to get in doesn't fit the club's aesthetics.

#  Enablement
Typically, there are performed via plugins.  APIServer has a feature called enable-admission-plugins that accepts the controller plugins as parameters.  This feature will be invoked prior to orchestrating, queueing, and/or modifying objects in a cluster.

