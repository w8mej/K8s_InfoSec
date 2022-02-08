# Network Security
Network security on a kubernetes stack may become extremely arcane and obtuse to learn, much less apply.  Not only does one have to deal with the data plane, but also manage the control plane, DNS, broadcasting, peering, etc....  The great part is that the networks are software-defined.  Some technologies are easy to pick up (Calico) while others (Istio) have a significant learning curve.  I will show the simpler side instead of causing one to scratch their head when they attempt to install Envoy on their Istio-enabled cluster.

# Default Deny
A great network security practice is to default deny traffic; then bless certain types, destinations, and sources of traffic.  One may apply k8s-defaultdeny_traffic.yml as an easy win.

# DNS?
One will observe the default deny policy disables DNS.  If one wishes to enable DNS (TCP and / or UDP), see k8s-DNS_UDP_TCP.yml - tweak it to your tcp or udp needs.  Then apply it.

# Inbound?
One will need to enable inbound traffic.  k8s-Ingress.yml is a great start.  Tweak it to your applicable namespace for the Ingress Controller 
