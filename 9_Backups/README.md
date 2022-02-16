# Backing up
"Pursue your dreams but have a backup plan."


"...It's backup day today so I'm pissed off. Being the BOFH, however, does have it's advantages. I reassign null to be the tape device - it's so much more economical on my time as I don't have to keep getting up to change tapes every 5 minutes. And it speeds up backups too, so it can't be all bad can it? Of course not...." - Bastard Operator From Hell
http://bofh.bjash.com/bofh/bofh1.html

# Kubernetes backups
There are MANY MANY ways to skin this cat.  For years, I used Heptio Ark.  However, I continue to change based upon the resources on hand.

## Heptio Ark
Now known as Velero, Velero has a client / server architecture.  Installing the server is straight forward.  Below is utilizing S3 as the data store

```
velero install \
 --provider aws \
 --plugins velero/velero-plugin-for-aws:v1.1.0,velero/velero-plugin-for-csi:v0.1.2  \
 --bucket bucket  \
 --secret-file minio.secret  \
 --use-volume-snapshots=true \
 --backup-location-config region=default,s3ForcePathStyle="true",s3Url=http://minio.mydomain.intra  \
 --snapshot-location-config region=default \
 --features=EnableCSI
 ```

 Snapshotting

 ```
 kubectl label VolumeSnapshotClass csi-rbdplugin-snapclass \
velero.io/csi-volumesnapshot-class=true

kubectl label VolumeSnapshotClass csi-cephfsplugin-snapclass \
velero.io/csi-volumesnapshot-class=true
```

Let's manually create a backup then set a cron job to run daily at 1 am machine local time

```
velero backup create nginx-backup \
--include-namespaces nginx-example --wait

velero backup describe nginx-backup
velero backup logs nginx-backup
velero backup get

velero schedule create nginx-daily --schedule="0 1 * * *" \
--include-namespaces nginx-example

velero schedule get
velero backup get
```

Instead, let's create a cluster policy that runs on a similar schedule

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: autobackup-policy
spec:
  background: false
  rules:
  - name: "add-velero-autobackup-policy"
    match:
        resources:
          kinds:
            - Namespace
          selector:
            matchLabels:
              nirmata.io/auto-backup: enabled
    generate:
        kind: Schedule
        name: "{{request.object.metadata.name}}-auto-schedule"
        namespace: velero
        apiVersion: velero.io/v1
        synchronize: true
        data:
          metadata:
            labels:
              nirmata.io/backup.type: auto
              nirmata.io/namespace: '{{request.object.metadata.name}}'
          spec:
            schedule: 0 1 * * *
            template:
              includedNamespaces:
                - "{{request.object.metadata.name}}"
              snapshotVolumes: false
              storageLocation: default
              ttl: 168h0m0s
              volumeSnapshotLocations:
                - default
```

Don't forget to test your backups continiously via CI / CD pipelines
```
kubectl delete ns nginx-example

velero restore create nginx-restore-test --from-backup nginx-backup
velero restore get

kubectl get po -n nginx-example
```

# Kanister
Another tool for backing up THE source of truth for a cluster (ETCD) - Kanister.  Straight forward when backed by S3.
```
kanctl create profile s3compliant --access-key <aws-access-key> \
        --secret-key <aws-secret-key> \
        --bucket <bucket-name> --region <region-name> \
        --namespace kanister

kubectl create secret generic etcd-details \
     --from-literal=cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --from-literal=cert=/etc/kubernetes/pki/etcd/server.crt \
     --from-literal=endpoints=https://127.0.0.1:2379 \
     --from-literal=key=/etc/kubernetes/pki/etcd/server.key \
     --from-literal=etcdns=kube-system \
     --from-literal=labels=component=etcd,tier=control-plane \
     --namespace kanister

kubectl label secret -n kanister etcd-details include=true
kubectl annotate secret -n kanister etcd-details kanister.kasten.io/blueprint='etcd-blueprint'
```

The template used by Kanister
```
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: etcd-blueprint
actions:
  backup:
    outputArtifacts:
      cloudObject:
        keyValue:
          backupLocation: "{{ .Phases.uploadSnapshot.Output.backupLocation }}"
    phases:
    - func: KubeTask
      name: takeSnapshot
      args:
        image: ghcr.io/kanisterio/kanister-kubectl:1.18
        command:
          - sh
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            export CACERT="{{ index .Object.data "cacert" | toString | b64dec }}"
            export CERT="{{ index .Object.data "cert" | toString | b64dec }}"
            export ENDPOINTS="{{ index .Object.data "endpoints" | toString | b64dec }}"
            export KEY="{{ index .Object.data "key" | toString | b64dec }}"
            export LABELS="{{ index .Object.data "labels" | toString | b64dec }}"
            export ETCDNS="{{ index .Object.data "etcdns" | toString | b64dec }}"

            ETCD_POD=$(kubectl get pods -n $ETCDNS -l $LABELS -ojsonpath='{.items[0].metadata.name}')

            kubectl exec -it -n $ETCDNS $ETCD_POD -c etcd -- sh -c "ETCDCTL_API=3 etcdctl --endpoints=$ENDPOINTS --cacert=$CACERT --cert=$CERT --key=$KEY snapshot save /tmp/etcd-backup.db"
            kando output etcdPod $ETCD_POD
            kando output etcdNS $ETCDNS

    - func: KubeTask
      name: uploadSnapshot
      args:
        image: ghcr.io/kanisterio/kanister-kubectl:1.18
        command:
          - sh
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            BACKUP_LOCATION=etcd_backups/{{ .Object.metadata.namespace }}/{{ toDate "2006-01-02T15:04:05.999999999Z07:00" .Time | date "2006-01-02T15:04:05Z07:00" }}/etcd-backup.db.gz
            kubectl cp {{ .Phases.takeSnapshot.Output.etcdNS }}/{{ .Phases.takeSnapshot.Output.etcdPod }}:/tmp/etcd-backup.db /tmp/etcd-backup.db
            gzip /tmp/etcd-backup.db
            kando location push --profile '{{ toJson .Profile }}'  /tmp/etcd-backup.db.gz --path $BACKUP_LOCATION
            kando output backupLocation $BACKUP_LOCATION

    - func: KubeTask
      name: removeSnapshot
      args:
        image: ghcr.io/kanisterio/kanister-kubectl:1.18
        command:
          - sh
          - -o
          - errexit
          - -o
          - pipefail
          - -c
          - |
            kubectl exec -it -n {{ .Phases.takeSnapshot.Output.etcdNS }} {{ .Phases.takeSnapshot.Output.etcdPod }} -c etcd -- sh -c "rm -rf  /tmp/etcd-backup.db"

  delete:
    type: Namespace
    inputArtifactNames:
    - cloudObject
    phases:
    - func: KubeTask
      name: deleteFromObjectStore
      args:
        namespace: "{{ .Namespace.Name }}"
        image: "ghcr.io/kanisterio/kanister-tools:0.32.0"
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          kando location delete --profile '{{ toJson .Profile }}' --path '{{ .ArtifactsIn.cloudObject.KeyValue.backupLocation }}'
```

As one can see, nothing really magical about it.  Time to schedule Kanister to run

```
kubectl create -n kanister -f -
apiVersion: cr.kanister.io/v1alpha1
kind: ActionSet
metadata:
  creationTimestamp: null
  generateName: backup-
  namespace: kanister
spec:
  actions:
  - blueprint: "<blueprint-name>"
    configMaps: {}
    name: backup
    object:
      apiVersion: v1
      group: ""
      kind: ""
      name: "<secret-name>"
      namespace: "<secret-namespace>"
      resource: secrets
    options: {}
    preferredVersion: ""
    profile:
      apiVersion: ""
      group: ""
      kind: ""
      name: "<profile-name>"
      namespace: kanister
      resource: ""
    secrets: {}
EOF

kubectl get actionsets
kubectl describe actionsets -n kanister backup-hnp95
```


## Restoration
You don't have backups until one can prove satisfactory restoration is possible.

```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --data-dir="/var/lib/etcd-from-backup" \
  --initial-cluster="ubuntu-s-4vcpu-8gb-blr1-01-master-1=https://127.0.0.1:2380" \
  --name="ubuntu-s-4vcpu-8gb-blr1-01-master-1" \
  --initial-advertise-peer-urls="https://127.0.0.1:2380" \
  --initial-cluster-token="etcd-cluster-1" \
  snapshot restore /tmp/etcd-backup.db
  ```


# Multi-tenant ETCD
when one starts architecting multiple ETCD nodes within a cluster, nearly the same steps are followed above.  One will need to choose the leader node one will use to restore the backup.  Then stop the static pods from all other leaer nodes.  This is accomplished by moving the static pod manifests from the static pod path (/etc/kubernetes/manifests.) Once this is accomplished, then restore the ETCD cluster on all leader nodes sequentially (1....10...)

An example below
```
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --data-dir="/var/lib/etcd-from-backup" \
  --initial-cluster="ubuntu-s-4vcpu-8gb-blr1-01-master-1=https://127.0.0.1:2380" \
  --name="ubuntu-s-4vcpu-8gb-blr1-01-master-1" \
  --initial-advertise-peer-urls="https://127.0.0.1:2380" \
  --initial-cluster-token="etcd-cluster-1" \
  snapshot restore /tmp/etcd-backup.db
  ```
  Where initial-cluster and name depends on the new leader node.  Then one will need to change the initial-cluster-token due to etcdctl restore creating a new member (better to have members with the same token then random members joining other clusters...)



  # Backup to Gitlab, BitBucket, or GitHub
  At the core of Kubernetes are a bunch of objects on a filesystem - interacted with via a HTTP API.  As a result, there exist multiple tools that will interact with these objects to create git repositories.  

  ## Kube-dump
  kube-dump is a simple utility for backing up partial to complete information on a kubernete's cluster.  Saving is done only for those resources to which you have read access.  One may pass a list of namespaces as an input, otherwise all available for your context will be used. Both namespace resources and global cluster resources are subject to persistence. One may use the utility locally as a regular script or run it in a container or in a kubernetes cluster, for example, as a CronJob.  It can create archives and rotate them after itself. Can commit state to git repository and push to remote repository. You can specify a specific list of cluster resources for unloading.

  ## Installation and usage
  ```
  ---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app: kube-dump
  name: kube-dump
  namespace: kube-dump

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-dump
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: kube-dump
    namespace: kube-dump
```

Setup via gitlab token

```
kubectl -n kube-dump create secret generic kube-dump \
  --from-literal=GIT_REMOTE_URL=https://oauth2:$TOKEN@haxx.gitlab.com/backups/k8s/cluster-0006.git
```
```
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  labels:
    app: kube-dump
  name: kube-dump
  namespace: kube-dump
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: kube-dump
          containers:
            - name: kube-dump
              image: woozymasta/kube-dump:latest
              imagePullPolicy: IfNotPresent
              envFrom:
                - secretRef:
                    name: kube-dump
              env:
                - name: MODE
                  value: "ns"
                - name: NAMESPACES
                  value: "dev,prod"
                - name: GIT_PUSH
                  value: "true"
                - name: GIT_BRANCH
                  value: "my-cluster"
              resources:
                limits:
                  cpu: 500m
                  memory: 200Mi
                requests:
                  cpu: 200m
                  memory: 100Mi
          restartPolicy: OnFailure
```
```
...
spec:
  schedule: "0 1 * * *"
...
              env:
                - name: MODE
                  value: "dump"
                - name: DESTINATION_DIR
                  value: "/data/dump"
                - name: GIT_PUSH
                  value: "true"
                - name: GIT_BRANCH
                  value: "master"
                - name: GIT_COMMIT_USER
                  value: "Kube Dump"
                - name: GIT_COMMIT_EMAIL
                  value: "automation@haxx.ninja"
                - name: GIT_REMOTE_URL
                  value: "git@haxx.gitlab.com:backups/k8s/cluster-0006_bkp.git"

kubectl apply -f cronjob-git-key.yaml -n kube-dump

mkdir -p ./.ssh
chmod 0700 ./.ssh
ssh-keygen -t ed25519 -C "kube-dump" -f ./.ssh/kube-dump
cat ./.ssh/kube-dump.pub

kubectl -n kube-dump create secret generic kube-dump-key \
  --from-file=./.ssh/kube-dump \
  --from-file=./.ssh/kube-dump.pub
```

Just that simple and sweet.  

