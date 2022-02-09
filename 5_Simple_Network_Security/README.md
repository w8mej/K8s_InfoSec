# Network Security
Network security on a kubernetes stack may become extremely arcane and obtuse to learn, much less apply.  Not only does one have to deal with the data plane, but also manage the control plane, DNS, broadcasting, peering, etc....  The great part is that the networks are software-defined.  Some technologies are easy to pick up (Calico) while others (Istio) have a significant learning curve.  I will show the simpler side instead of causing one to scratch their head when they attempt to install Envoy on their Istio-enabled cluster.

# Default Deny
A great network security practice is to default deny traffic; then bless certain types, destinations, and sources of traffic.  One may apply k8s-defaultdeny_traffic.yml as an easy win.  If only for operational uptime requirements balancing blast radius, it is important to enforce separation of containers. As you can see below, one may create a NetworkPolicy for a specific namespace and/or the entire cluster. So don’t forget to create the default best practice Policies. 

# DNS?
One will observe the default deny policy disables DNS.  If one wishes to enable DNS (TCP and / or UDP), see k8s-DNS_UDP_TCP.yml - tweak it to your tcp or udp needs.  Then apply it.

# Inbound?
One will need to enable inbound traffic.  k8s-Ingress.yml is a great start.  Tweak it to your applicable namespace for the Ingress Controller.  Here is an example of tcp 9093 tcp inbound for the application labelled prometheus in the monitoring namespace called alertmanager-mesh

```
apiVersion: extensions/v1beta1
kind: NetworkPolicy
metadata:
  name: alertmanager-mesh
  namespace: monitoring
spec:
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: prometheus
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9093
      protocol: tcp
  podSelector:
    matchLabels:
      app: alertmanager
```

Or to allow anything inbound to prometheus over tcp 9090
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus
  namespace: monitoring
spec:
  policyTypes:
  - Ingress
  podSelector:
    matchLabels:
      app: prometheus
  ingress:
  - from:
    - podSelector: {}
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9090
```


# Egress traffic / outbound traffic policy management
Same same, different, but still same.

Here is an example called allow-egress-same-namespace that allows blue labelled apps in the default namespace outboud on port 80
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-egress-same-namespace
  namespace: default
spec:
  policyTypes:
    - Egress
  podSelector:
    matchLabels:
      color: blue
  egress:
  - to:
    - podSelector:
        matchLabels:
          color: red
    ports:
    - port: 80
```

or if you wish to be insecure, allow all traffic within a namespace via
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
spec:
  policyTypes:
  - Ingress
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

Or maybe you have a prod namespace app labelled orders that needs to connect to an external mysql database (port 3306) in an RFC 1918 IP space.
```
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: customer-api-allow-web
  namespace: prod
spec:
  policyTypes:
  - Egress
  podSelector:
    matchLabels:
      app: orders
  egress:
  - ports:
    - port: 3306
    to:
    - ipBlock:
        cidr: 172.16.32.0/27
```


