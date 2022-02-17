# Rotating certificates 
On a live kubernetes cluster, rotating certificates is a challenging task.  There are many different mechanisms to perform these functions but none work exceeding well with predictable outcomes like water flowing out of a tap.  By default, a typical kubernetes stack will generate certificates, including the signing and CA, for a 1 year life span.

## Sanity checks
# Which certificates are expiring when?  May want to create a ticket to handle these
```
kubeadm certs check-expiration
```
# Or manually verify the certificate's expiration
```
find /etc/kubernetes/pki/ -type f -name "*.crt" -print |xargs -L 1 -t  -i bash -c 'openssl x509  -noout -text -in {}|grep Not'
find /var/lib/kubelet/pki/ -type f -name "*.crt" -print |xargs -L 1 -t  -i bash -c 'openssl x509  -noout -text -in {}|grep Not' 
```
## To allow the stack to auto-rotate the certificates upon restarts, here are a few roles to add to users and actions
# Setup the permissions
```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
kubectl create clusterrolebinding node-client-auto-approve-csr --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient --group=system:node-bootstrappers
kubectl create clusterrolebinding node-client-auto-renew-crt --clusterrole=system:certificates.k8s.io:certificatesigningrequests:selfnodeclient --group=system:nodes
```
# Configure kubelet's rotation
```
vim /etc/systemd/system/kubelet.env
```
# Or
```
vim /var/lib/kubelet/kubeadm-flags.env
```

# To add the following additional arguments
```
KUBELET_EXTRA_ARGS=="--rotate-certificates=true --rotate-server-certificates=true"
```

# Restart kubelet
```
systemctl restart kubelet
```

# Or if one wishes to use kubeadm
```
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

# OR
```
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```
# To add the following additional arguments
```
Environment="KUBELET_EXTRA_ARGS=--rotate-certificates=true --rotate-server-certificates=true"
```

# Reload the services and restart kubelet
```
systemctl daemon-reload
systemctl restart kubelet
```
# Sanity check kubelet has the runtime parameter - rotate certificates
```
ps -ef | grep kubelet | grep "rotate-certificates"
```
# Sanity check the certificate signing requests are handled appropriately and approved
```
kubectl get csr
```

## Manual renewal
For those times one requires to manually rotate a certificate due to a breach, unauthorized etc file store access, etc... one is able to rotate the certificates.  Must be performed from all control planes.  Here is an example to rotate all certs in the alpha namespace
```
sudo kubeadm alpha certs renew all
```
# Remember this approach relies upon restarting the cluster.  
If one doesn't restart the cluster every 60-90 days, then the above monitoring for expired certificates attached to a jira ticket or google calendar event is an appropriate compensating control.

