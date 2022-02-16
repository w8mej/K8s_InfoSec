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


