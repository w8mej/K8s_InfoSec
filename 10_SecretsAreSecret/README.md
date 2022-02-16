# Kubernetes Secrets are not Secret
Due to personal engineering and Google's internal secrets manager; there was no material desire nor want to build and introduce a proper secrets management featureset into or onto Kubernetes.  As a result, the secrets are stored as base64 objects in the ETCD files as well as exposed (per policy in namespace and cluster) via ETCD's API.  As a result, the simplest and quickest deployment method (for now) is to operate a secrets mgt. solution outside of Kubernetes.  To quote Kubernetes Security SIG chair - "It was not designed to be secure." It being secrets management.

## For Now
There are techniques to utilize Kubernete's secrets features as middleware, opaquely driven by Hashicorp Vault, AWS Secrets Manager, AWS KMS, AWS CloudHSM, etc...  I am going to skip on those as this is a great time to introduce side car patterns.  This pattern is commonly found when attempting to intrument workloads to perform objectives while controls are applied transparently.

# Hashicorp Vault
HashiCorp Vault is a secrets management solution that brokers access for both humans and machines, through programmatic access, to systems. Secrets can be stored, dynamically generated, dynamically rotated, and in the case of encryption, keys can be consumed as a service without the need to expose the underlying key materials.

## Insecure Architecture
Here is an insecure architecture to allow one to play with this approach.  Hardening the architecture is left as an exercise to the reader
![Vault Kubernetes](IntroArchitectureInSecure.png)

## Installation
To install Vault, find the appropriate package for your system and download it. Vault is packaged as a zip archive.  https://www.vaultproject.io/downloads .  After downloading Vault, unzip the package. Vault runs as a single binary named vault. Any other files in the package can be safely removed and Vault will still function.  The final step is to make sure that the vault binary is available on the PATH. See this page for instructions on setting the PATH on Linux and Mac https://stackoverflow.com/questions/14637979/how-to-permanently-set-path-on-linux . This page contains instructions for setting the PATH on Windows - https://stackoverflow.com/questions/1618280/where-can-i-set-path-to-make-exe-on-windows .

After installing Vault, verify the installation worked by opening a new terminal session and checking that the vault binary is available. By executing vault, you should see help output.  If you get an error that the binary could not be found, then your PATH environment variable was not setup properly. Please go back and ensure that your PATH variable contains the directory where Vault was installed.  Otherwise, Vault is installed and ready to go!

## Starting the Server
Vault operates as a client/server application. The Vault server is the only piece of the Vault architecture that interacts with the data storage and backends. All operations done via the Vault CLI interact with the server over a TLS connection.  First, start a Vault dev server. The dev server is a built-in, pre-configured server that is not very secure but useful for playing with Vault locally.   To start the Vault dev server, run:
```
vault server -dev
```

Notice that Unseal Key and Root Token values are displayed.  The dev server stores all its data in-memory (but still encrypted), listens on localhost without TLS, and automatically unseals and shows you the unseal key and root access key.  Do not run a Vault dev server in production. This approach is only used here to simplify the unsealing process for this demonstration.  
With the dev server started, perform the following: Launch a new terminal session.  Copy and run the export VAULT_ADDR ... command from the terminal output. This will configure the Vault client to talk to the dev server.  Save the unseal key somewhere. Don't worry about how to save this securely. For now, just save it anywhere.  Set the VAULT_TOKEN environment variable value to the generated Root Token value displayed in the terminal output.  To interact with Vault, you must provide a valid token. Setting this environment variable is a way to provide the token to Vault via CLI.  Verify the server is running by running the vault status command. If it ran successfully, the output should look like the following:
```
vault status

Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.7.0
Storage Type    inmem
Cluster Name    vault-cluster-4d862b44
Cluster ID      92143a5a-0566-be89-f229-5a9f9c47fb1a
HA Enabled      false
```

If the output looks different, restart the dev server and try again. 

## Insecure configuration for Kubernetes
Let's setup key value secrets.  We will use kv storage to handle the application's secrets as well as the clusters' secrets and config.
```
vault secrets enable kv

vault kv put kv/secret/helloworld/config username='winning' password='picklerick'

vault kv get -format=json kv/secret/helloworld/config | jq ".data"
```

Now let's create a Vault policy to enable one to read the secrets as well as list the secret's metadata behind the helloworld kv store.
```
vault policy write helloworld-kv-ro - <<EOF
path "kv/secret/helloworld/*" {
    capabilities = ["read", "list"]
}
EOF
```

Now let's create and apply the service account
```
kubectl apply --filename ./vault-injector-serviceaccount.yml
```

Here is where a lot of the insecurity comes from - poor cryptographic materials handling.  Let's grab the relevant tokens and cryptographic materialsand enable Vault to allow Kubernete's auth.
```
export EXTERNAL_VAULT_ADDR=192.168.1.1
export K8S_HOST=192.168.1.243

export VAULT_SA_NAME=$(kubectl get sa helloworld -o jsonpath="{.secrets[*]['name']}")

export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)

export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

vault auth enable kubernetes
```

Let's write the kubernetes config and apply the appropriate vault policy to the helloworld role
```
vault write auth/kubernetes/config issuer="https://kubernetes.default.svc.cluster.local" token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="https://$K8S_HOST:6443" kubernetes_ca_cert="$SA_CA_CRT"

vault write auth/kubernetes/role/helloworld bound_service_account_names=helloworld bound_service_account_namespaces=default policies=helloworld-kv-ro ttl=24h
```

### Vault Agent Injector / Sidecar Init Container
The Vault Agent Injector alters pod specifications to include Vault Agent containers that render Vault secrets to a shared memory volume using Vault Agent Templates. By rendering secrets to a shared volume, containers within the pod can consume Vault secrets without being Vault aware.  The injector is a Kubernetes Mutation Webhook Controller. The controller intercepts pod events and applies mutations to the pod if annotations exist within the request. This functionality is provided by the vault-k8s project and can be automatically installed and configured using the Vault Helm chart.

The Vault Agent Injector works by intercepting pod CREATE and UPDATE events in Kubernetes. The controller parses the event and looks for the metadata annotation vault.hashicorp.com/agent-inject: true. If found, the controller will alter the pod specification based on other annotations present.  At a minimum, every container in the pod will be configured to mount a shared memory volume. This volume is mounted to /vault/secrets and will be used by the Vault Agent containers for sharing secrets with the other containers in the pod.  Next, two types of Vault Agent containers can be injected: init and sidecar. The init container will prepopulate the shared memory volume with the requested secrets prior to the other containers starting. The sidecar container will continue to authenticate and render secrets to the same location as the pod runs. Using annotations, the initialization and sidecar containers may be disabled.  Last, two additional types of volumes can be optionally mounted to the Vault Agent containers. The first is secret volume containing TLS requirements such as client and CA (certificate authority) certificates and keys. This volume is useful when communicating and verifying the Vault server's authenticity using TLS. The second is a configuration map containing Vault Agent configuration files. This volume is useful to customize Vault Agent beyond what the provided annotations offer.

The primary method of authentication with Vault when using the Vault Agent Injector is the service account attached to the pod. Other authentication methods can be configured using annotations.  For Kubernetes authentication, the service account must be bound to a Vault role and a policy granting access to the secrets desired.  A service account must be present to use the Vault Agent Injector with the Kubernetes authentication method. It is not recommended to bind Vault roles to the default service account provided to pods if no service account is defined.

Installation is as simple as 
```
helm install vault hashicorp/vault --set "injector.externalVaultAddr=http://$EXTERNAL_VAULT_ADDR:8200"
```



# Deploying Hello World
It is simple as deploying the workload and its' sidecar container
```
kubectl apply -f helloworld.yml
```

Then verify the secrets and everything looks ok

```
kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") --container hello -- cat /vault/secrets/config.txt
```


Enjoy.  As mentioned above, the exercise to secure Vault or to use native Kubernete's secrets management backed by Vault are exercises for the reader to learn and apply.


