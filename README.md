# VolSync Lab with RHACM

## Introduction
This tutorial is a step by step guide to show the implementation of VolSync with Red Hat Advanced Cluster for Kubernetes 
on AWS infrastructure. See the related [documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.0/html/manage_cluster/creating-a-cluster-with-red-hat-advanced-cluster-management-for-kubernetes#creating-a-cluster-on-amazon-web-services) to learn how to
create a new OpenShift cluster in RHACM in a fully unattended approach.

## Project and documentation links:
- [VolSync Home](https://volsync.readthedocs.io/en/latest/index.html)
- [VolSync on GitHub](https://github.com/backube/volsync)
- [RHACM documentation for VolSync](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.4/html/clusters/managing-your-clusters#volsync)
- [Rsync example across two clusters](https://volsync.readthedocs.io/en/latest/usage/rsync/multi_context_sync_cli.html)

## Step 1: Create managed clusters
Create two managed clusters on RHACM using AWS provider. For the purpose of this lab the two clusters will be named
`ocp-src` and `ocp-dst`

Once created, label abel clusters with a common label that will be used by the policy PlacementRule.

```
$ kubectl label managedcluster ocp-src env=volsync-lab
$ kubectl label managedcluster ocp-dst env=volsync-lab
```

Save clusters context for future use:
```
$ oc login -u kubeadmin -p <PASSWORD> <SRC_CLUSTER_API>
$ SRC_CTX=$(kubectl config current-context) 

$ oc login -u kubeadmin -p <PASSWORD> <DEST_CLUSTER_API>
$ DST_CTX=$(kubectl config current-context) 
```

## Step 2: Create the RHACM Policy
Create the VolSync policy on RHACM. The policy deploys VolSync subscription on the Hub cluster and creates the resources on clusters targeted by the PlacementRule.

Edit the policy file and update the PlacementRule to match the lab clusters labels:
```
spec:
  clusterSelector:
    matchLabels:
      env: volsync-lab
```

Once updated, create the Policy resource.
```
$ kubectl create -f acm_policy/policy-persistent-data-management.yaml
```

After the policy enforcement, a deployment named `volsync` in the `volsync-system` namespace will appear in the managed clusters.
The generated pod executes the VolSync manager container, which runs the ReplicationDestination and ReplicationSource controllers, as well as
the volume handlers and movers.

## Step 3: Configure CSI StorageClasses
Annotate the AWS `gp2-csi` StorageClass to become the default. At the same time modify the annotation of the non-CSI `gp2` StorageClass.
Repeat the same action on both source and destination clusters.

```
$ kubectl annotate sc/gp2 storageclass.kubernetes.io/is-default-class="false" --overwrite --context $SRC_CTX
$ kubectl annotate sc/gp2-csi storageclass.kubernetes.io/is-default-class="true" --overwrite --context $SRC_CTX

$ kubectl annotate sc/gp2 storageclass.kubernetes.io/is-default-class="false" --overwrite --context $DST_CTX
$ kubectl annotate sc/gp2-csi storageclass.kubernetes.io/is-default-class="true" --overwrite --context $DST_CTX
```

### Optional: Configure VolumeSnapshotClass
Verify if a VolumeSnapshotClass exists. If available a VSC is already available **skip this subsection**, otherwise create a new one.
The following example is a VSC for AWS:

```
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: gp2-csi
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

Create the VSC on both clusters:
```
$ kubectl create -f src_cluster/volumesnapshotclass.yaml --context $SRC_CTX
$ kubectl create -f dest_cluster/volumesnapshotclass.yaml --context $DST_CTX
```

Also annotate the VSC as the default:
```
$ kubectl annotate volumesnapshotclass/gp2-csi snapshot.storage.kubernetes.io/is-default-class="true" --context $SRC_CTX
$ kubectl annotate volumesnapshotclass/gp2-csi snapshot.storage.kubernetes.io/is-default-class="true" --context $DST_CTX
```

## Step 4: Create SSH keys Secrets
Create the public keys.
Generate a keypair for destination cluster:

```
$ ssh-keygen -t rsa -b 4096 -f destination -C "" -N ""
```

Generate a keypair for source cluster:

```
$ ssh-keygen -t rsa -b 4096 -f source -C "" -N ""
```

Create a secret for the destination cluster. 
The destination needs access to the public and private destination keys but only the public source key:

```
$ kubectl create ns volsync-lab-dst --context $SRC_CTX
$ kubectl create secret generic volsync-rsync-dest-dest-app-volume-dest --from-file=destination=destination --from-file=source.pub=source.pub --from-file=destination.pub=destination.pub -n volsync-lab-dst --context $SRC_CTX
```

Create a secret for the source cluster:

```
$ kubectl create ns volsync-lab-src --context $DST_CTX
$ kubectl create secret generic volsync-rsync-dest-src-app-volume-dest --from-file=source=source --from-file=source.pub=source.pub --from-file=destination.pub=destination.pub -n volsync-lab-src --context $DST_CTX
```

## Step 5: Create an example stateful app
Create an example stateful database (MySQL pod) with its own PVC.

```
$ kubectl create -f src_cluster/source_database/ --context $SRC_CTX
```

Update the content of the example database:
```
$ kubectl --context $SRC_CTX exec --stdin --tty -n volsync-lab-src `kubectl --context $SRC_CTX get pods -n volsync-lab-src | grep mysql | awk '{print $1}'` -- /bin/bash
# mysql -u root -p$MYSQL_ROOT_PASSWORD
> create database my_new_database;
> show databases;
> exit
$ exit
```

## Step 6: Create the ReplicationDestination on the target cluster
Create the ReplicationDestination on the target cluster. The following example shows the custom ReplicationDestination used in this demo:
```
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: app-volume-dest
  namespace: volsync-lab-dst
spec:
  rsync:
    serviceType: LoadBalancer
    copyMethod: Snapshot
    capacity: 10Gi
    accessModes: ["ReadWriteOnce"]
    sshKeys: volsync-rsync-dest-dest-app-volume-dest
```

Create the resource on the destination cluster:
```
$ kubectl create -f dest_cluster/replicationdestination.yaml --context $DST_CTX
```

Since the replication must be across two clusters the serviceType must be set to LoadBalancer to allow remote access to the
exposed SSH service.
Also notice the usage of the previously defined secret `volsync-rsync-dest-dest-app-volume-dest` in the `.spec.rsync.sshKeys` key.

After successful creation of the ReplicationDestination, take note of the `.status.rsync.address` field: this is the URL of the
LoadBalancer service that will expose the SSH service used by rsync.
```
status:
  rsync:
      address: a7d6b3795004742d7bebb9c89a91de40-832ef8cb58659420.elb.eu-central-1.amazonaws.com
```

## Step 7: Create the ReplicationSource on the source cluster
Create the ReplicationSource on the source cluster:

```
$ kubectl create -f replicationsource.yaml --context source-cluster
```

```
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: app-volume-src
  namespace: volsync-lab
spec:
  sourcePVC: mysql-pv-claim
  trigger:
    schedule: "*/5 * * * *"
  rsync:
    sshKeys: volsync-rsync-dest-src-app-volume-dest
    address: a7d6b3795004742d7bebb9c89a91de40-832ef8cb58659420.elb.eu-central-1.amazonaws.com
    copyMethod: Snapshot
```

Since the gp2-csi storageclass does not support Cloning (the default copy method), change the `.spec.rsync.copyMethod` to `Snapshot`.

## Step 8: Verify volume synchronization

Verify that the volume is synchronized on the source cluster:

```
$ oc get ReplicationSource -n volsync-lab-src --context $SRC_CTX 
NAME             SOURCE           LAST SYNC              DURATION        NEXT SYNC
app-volume-src   mysql-pv-claim   2022-01-20T00:40:27Z   27.118988169s   2022-01-20T00:45:00Z
```

Check that the ReplicationDestination is synchronized on the destination cluster:
```
$ oc get ReplicationDestination -n volsync-lab-dst --context $DST_CTX 
NAME              LAST SYNC              DURATION         NEXT SYNC
app-volume-dest   2022-01-20T00:40:26Z   5m0.482863439s
```

Verify that a VolumeShapshot exists and is updated on the destination cluster:
```
$ oc get volumesnapshot -n volsync-lab-dst --context $DST_CTX
NAME                                          READYTOUSE   SOURCEPVC                      SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS   SNAPSHOTCONTENT                                    CREATIONTIME   AGE
volsync-dest-app-volume-dest-20220120004026   true         volsync-dest-app-volume-dest                           10Gi          gp2-csi         snapcontent-92cbff78-f704-4c17-9875-812755dbf5cf   2m53s          2m53s
```

## Step 9: Restore an application on the target cluster

It's finally time to test an application restore on the destination cluster. We will create a second database instance that
will consume a PVC that uses the snapshot as a data source.

A PVC bound to a snapshot has a slightly different syntax, like in the following example:
```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: volsync-dest-app-volume-dest
spec:
  accessModes:
    - ReadWriteOnce
  dataSource:
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
    name: <REPLACE_WITH_SNAPSHOT_NAME>
  resources:
    requests:
      storage: 10Gi
```

The `.spec.dataSource` field defins the snapshot name to be used. Update its value with the latest snapshot a
and create the PVC on the destination cluster.

```
$ kubectl create -f dest_cluster/dest_database/mysql-pvc.yaml -n volsync-lab-dst --context $DST_CTX
```
Inspect the database content to see if the volume has correctly replicated:


```
$ kubectl --context $DST_CTX exec --stdin --tty -n volsync-lab-dst `kubectl --context $DST_CTX get pods -n volsync-lab-dst | grep mysql | awk '{print $1}'` -- /bin/bash
# mysql -u root -p$MYSQL_ROOT_PASSWORD
> show databases;
> exit
$ exit
```

