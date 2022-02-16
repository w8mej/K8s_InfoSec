# Kubernetes Secrets are not Secret
Due to personal engineering and Google's internal secrets manager; there was no material desire nor want to build and introduce a proper secrets management featureset into or onto Kubernetes.  As a result, the secrets are stored as base64 objects in the ETCD files as well as exposed (per policy in namespace and cluster) via ETCD's API.  As a result, the simplest and quickest deployment method (for now) is to operate a secrets mgt. solution outside of Kubernetes.  To quote Kubernetes Security SIG chair - "It was not designed to be secure." It being secrets management.

## For Now
There are techniques to utilize Kubernete's secrets features as middleware, opaquely driven by Hashicorp Vault, AWS Secrets Manager, AWS KMS, AWS CloudHSM, etc...  I am going to skip on those as this is a great time to introduce side car patterns.  This pattern is commonly found when attempting to intrument workloads to perform objectives while controls are applied transparently.

# Hashicorp Vault
HashiCorp Vault is a secrets management solution that brokers access for both humans and machines, through programmatic access, to systems. Secrets can be stored, dynamically generated, dynamically rotated, and in the case of encryption, keys can be consumed as a service without the need to expose the underlying key materials.

## Insecure Architecture
Here is an insecure architecture to allow one to play with this approach.  Hardening the architecture is left as an exercise to the reader
![Vault Kubernetes](IntroArchitectureInSecure.png)
