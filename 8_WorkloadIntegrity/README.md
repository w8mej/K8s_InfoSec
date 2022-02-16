# Deploying workloads with Integrity with Assurance
Utilizing the Admission Controller technique, one is able to utilize a plugin/bundle that integrates container image verification and assurance checking utilizing cryptographic techniques such as hashing and signing.

# Prereq - Notary
A popular project undergoing its' phoenix rebirth is called Notary.  It is an open source tool for signing contained based upon The Update Framework.  All signing and cryptographic key hierarchy is based upon TUF.  5 keys exist to sign metadata files, sizes, and related cryptographic hashes.

## Creating a signed image using Notary
Really simple walkthrough follows.  In no way are the keys being handled here handled in a secure manner.  They are being used to demonstrate the intent and spirit of image signing.  Remember, never sign the images where the keys are stored, and always handle in a cryptographically, formally correct manner.

```
notary -s https://notary.docker.io -d ~/.docker/trust init -p docker.io/projectnexus/jq     
Root key found, using: 234234.......
Enter passphrase for root key with ID 4232: 
Enter passphrase for new targets key with ID 542: 
Repeat passphrase for new targets key with ID 7657: 
Enter passphrase for new snapshot key with ID 73645: 
Repeat passphrase for new snapshot key with ID 922345: 
Enter username: {winninguser}
Enter password: 
Auto-publishing changes to docker.io/projectnexus/jq
Enter username: {winnineruser}
Enter password: 
Successfully published changes for repository docker.io/projectnexus/jq


export DOCKER_CONTENT_TRUST=1
export DOCKER_CONTENT_TRUST_SERVER=https://notary.docker.io
docker tag alpine:latest projectnexus/jq:signed
docker push projectnexus/jq:signed
```

When one examines the file system structure of the newly created signed image, one will see the following information

```
$ find ~/.docker/trust/ | head
/home/projectnexus/.docker/trust/
/home/projectnexus/.docker/trust/private
/home/projectnexus/.docker/trust/private/1f4a9a0922605b3bc19c97e180d962d530721288f4fd0845ad0aa37ba4a6f95d.key
/home/projectnexus/.docker/trust/private/fe30e72f5976b2ae7d0d365f28dacfae9c71f11ad854065603ccc806900e84fa.key
/home/projectnexus/.docker/trust/private/3da0d27e2d3b964d238d1d184c7578b5f2737b918ec5b8265474e22b07b2ea22.key
/home/projectnexus/.docker/trust/private/root-priv.key
/home/projectnexus/.docker/trust/private/root-pub.pem
/home/projectnexus/.docker/trust/tuf
/home/projectnexus/.docker/trust/tuf/docker.io
/home/projectnexus/.docker/trust/tuf/docker.io/projectnexus
```


# Assurance and verification gatekeeper - Connaisseur
A simple to demonstrate tool called Connaisseur is great to add into the Admission Controller middleware to execute the container image verification workload.    Let's start off creating the various cryptographic artifacts.  As mentioned above, these artifacts are handled for demonstration purposes, not to be utilized in Production workflows and environments.

```
cd ~/.docker/trust/private
sed '/^role:\sroot$/d' $(grep -iRl "role: root" .) > root-priv.key
openssl ec -in root-priv.key -pubout -out root-pub.pem
```

Now let's install the helm chart for Connaisseur - ensuring to update it with the relevant artifact information - if no validator is configured, default to be used.  notaryv2 support is experimental so let's try cosign or notaryv1.  host is the notary server to send the webhook request to.  key - the artifact used to verify notary's signatures.

```
...
validators:
...
# static validator that allows each image
- name: allow
  type: static
  approve: true
# pre-configured nv1 validator for public notary from Docker Hub
- name: dockerhub_basics
  type: notaryv1
  host: notary.docker.io
  trust_roots:
    # public key for official docker images (https://hub.docker.com/search?q=&type=image&image_filter=official)
    # !if not needed feel free to remove the key!
  - name: docker_official
    key: |
      -----BEGIN PUBLIC KEY-----
      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEOXYta5TgdCwXTCnLU09W5T4M4r9f
      QQrqJuADP6U7g5r9ICgPSmZuRHP/1AYUfOQW3baveKsT969EfELKj1lfCA==
      -----END PUBLIC KEY-----      
  # public key pwnage repo including Connaisseur images
  # !this key is critical for Connaisseur!
  - name: pwnage_official
    key: |
      -----BEGIN PUBLIC KEY-----
      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEsx28WV7BsQfnHF1kZmpdCTTLJaWe
      d0CA+JOi8H4REuBaWSZ5zPDe468WuOJ6f71E7WFg3CVEVYHuoZt2UYbN/Q==
      -----END PUBLIC KEY-----      
    # public key pwnage repo including projectnexus images
  - name: projectnexus_official
    key: |
      -----BEGIN PUBLIC KEY-----
      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE9m6WfwViwT8lYjLF6jAs1bvd1hPp
      cRUmONP49JszW1X/6Q22DygylIJGyC8IXeb3zBWVMoYDxauiqrFomHUOEA==
      -----END PUBLIC KEY-----      

policy:
- pattern: "*:*"
- pattern: "docker.io/library/*:*"
  validator: dockerhub_basics
  with:
    trust_root: docker_official
- pattern: "k8s.gcr.io/*:*"
  validator: allow
- pattern: "docker.io/pwnage/*:*"
  validator: dockerhub_basics
  with:
    trust_root: pwnage_official
- pattern: "docker.io/projectnexus/*:*"
  validator: dockerhub_basics
  with:
    trust_root: projectnexus_official
```


Deploy

```
kubectl get all -n connaisseur
NAME                                          READY   STATUS    RESTARTS   AGE
pod/connaisseur-deployment-565d45bb74-ktbmb   1/1     Running   0          71s
pod/connaisseur-deployment-565d45bb74-pfghx   1/1     Running   0          71s
pod/connaisseur-deployment-565d45bb74-rcj44   1/1     Running   0          71s

NAME                      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/connaisseur-svc   ClusterIP   10.43.196.6   <none>        443/TCP   71s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/connaisseur-deployment   3/3     3            3           71s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/connaisseur-deployment-565d45bb74   3         3         3       71s
```

That is it.  Short, simple, and sweet.

# Assurance testing failure
Let's deploy an unauthorized (as determined by signature) image.

```
kubectl run unsigned --image=docker.io/projectnexus/jq:unsigned
Error from server: admission webhook "connaisseur-svc.connaisseur.svc" denied the request: Unable to find signed digest for image docker.io/projectnexus/jq:unsigned.

kubectl run signed --image=docker.io/projectnexus/jq:signed
pod/signed created

kubectl get po
```



# Another option - Kyberno and Cosign
## Prereq - Cosign
With assistance from Linux Foundation Signstore's team, Google created Cosign.  Engineers and Analysts do not need to worry about the cryptographic underpinnings of their work.  It should be inherently built into the infrastructure, transparent to their needs.  It just works like water flowing out of the tap.  With image signing by Cosign, one doesn't need to change the infrastructure to store public keys.  The key's signatures are tags of the image linked to the associated image via digest.  Cosign supports a variety of key providers
```
fixed, text-based keys generated using cosign generate-key-pair
cloud KMS-based keys generated using cosign generate-key-pair -kms
keys generated on hardware tokens using the PIV interface using cosign piv-tool
kubernetes-secret based keys generated using cosign generate-key-pair -k8s
```

## Signing an image
Yes, it is this easy.
```
docker pull alpine:edge
docker tag alpine:edge projectnexus/jq:unsigned
docker push projectnexus/jq:unsigned


docker pull alpine:latest
docker tag alpine:latest projectnexus/jq:cosign
docker push projectnexus/jq:cosign


cosign sign -key ~/data/cosign.key projectnexus/jq:cosign
Enter password for private key: 
Pushing signature to: index.docker.io/projectnexus/jq:sha256-.....sig
```

Verification is just as easy
```
cosign verify -key ~/data/cosign.pub projectnexus/jq:cosign

Verification for projectnexus/jq:cosign --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
{"critical":{"identity":{"docker-reference":"index.docker.io/projectnexus/jq"},"image":{"docker-manifest-digest":"sha256:..."},"type":"cosign container image signature"},"optional":null}
```

# Kyverno
Installation is straight forward.  All one needs to utilize is the verifyImage's feature.  The default webhook http timeout is expecting a low latency, highly responsive endpoint.  One will need to correct it to a more humane timeout.
```
kubectl patch mutatingwebhookconfigurations kyverno-resource-mutating-webhook-cfg \
--type json \
-p='[{"op": "replace", "path": "/webhooks/0/failurePolicy", "value": "Ignore"},{"op": "replace", "path": "/webhooks/0/timeoutSeconds", "value": 15}]'
```

## Cluster policy
The cluster policy is straight forward for verifying all images
```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: enforce
  background: false
  rules:
    - name: check-image
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
      - image: "docker.io/projectnexus/jq:*"
        key: |-
          -----BEGIN PUBLIC KEY-----
          ...
          -----END PUBLIC KEY----- 
```

Deploying is as simple as 
```
kubectl create deployment signed-my \
--image=projectnexus/jq:cosign
```

## Assurance testing failure
Deploying an unsigned container results should result in failure akin to what we observe below;
```
error: failed to create deployment: admission webhook "mutate.kyverno.svc" denied the request: 

resource Deployment/kyverno-system/unsigned-my was blocked due to the following policies

verify-image:
  autogen-verify-image: 'image verification failed for docker.io/projectnexus/jq:unsigned:
    failed to verify image: fetching signatures: getting signature manifest: GET https://index.docker.io/v2/projectnexus/jq/manifests/sha256-....sig:
    MANIFEST_UNKNOWN: manifest unknown; map[Tag:sha256-.....sig]'
```


