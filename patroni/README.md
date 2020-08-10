# Patroni
Patroni is a template for you to create your own customized, high-availability solution using Python and for maximum accessibility 
a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. 
Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly 
deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful.


## Ceph Storage Class
Rook operator is installed to provide ceph storage-class. PVC is created which internally creates PV coming from ceph because of it's `VOLUMEBINDINGMODE`. The pgbench pod is spawned 
which makes use of that PVC. Velero has csi plugins registered to it because of which it creates csi snapshot on using commands `oc create -f postgres-backup.yaml` and `oc create -f postgres-restore.yaml`.

- Ceph is used as the default storageclass which helps velero to take csi snapshots of backups. Check if velero has csi plugins registered to it by this command,

```
$ velero get plugins | grep "csi"
velero.io/csi-pvc-backupper                            BackupItemAction
velero.io/csi-volumesnapshot-backupper                 BackupItemAction
velero.io/csi-volumesnapshotclass-backupper            BackupItemAction
velero.io/csi-volumesnapshotcontent-backupper          BackupItemAction
velero.io/csi-pvc-restorer                             RestoreItemAction
velero.io/csi-volumesnapshot-restorer                  RestoreItemAction
velero.io/csi-volumesnapshotclass-restorer             RestoreItemAction
velero.io/csi-volumesnapshotcontent-restorer           RestoreItemAction
```

<b>Note:</b> Run the following command `oc edit volumesnapshotclass` and change the field `deletionPolicy` to `Retain`.

```
apiVersion: snapshot.storage.k8s.io/v1beta1
deletionPolicy: Retain
driver: rook-ceph.rbd.csi.ceph.com
kind: VolumeSnapshotClass
```

## Installation
The setup assumes you have installed velero in oadp-operator namespace. If velero is installed in other namespace, 
change the `namespace` field in `postgres-backup.yaml` and `postgres-restore.yaml` files. 

<b>Note:</b> For allowing pods to reference `template_patroni_persistent.yaml` across projects 
follow the steps in the [link](https://docs.openshift.com/online/starter/openshift_images/managing_images/using-image-pull-secrets.html#images-allow-pods-to-reference-images-across-projects_using-image-pull-secrets).

To install patroni run the following commands,

```
oc new-project patroni
oc new-build https://github.com/konveyor/velero-examples --context-dir=patroni --name=patroni -n patroni
oc create -f template_patroni_persistent.yaml -n patroni
oc new-app patroni-pgsql-persistent 
oc create -f pgbench.yaml
```

The command `oc new-build https://github.com/konveyor/velero-examples --context-dir=patroni --name=patroni -n patroni` pushes the image to internal image registry. To check that, run the following commands,

```
$ oc get istag
NAME             IMAGE REFERENCE                                                                                                                            UPDATED
patroni:latest   image-registry.openshift-image-registry.svc:5000/patroni/patroni@sha256:d879c4f6502cc48b69d4aceefa4ff166b2900ff8d11b30937c59da20e3711aa5   44 seconds ago
```
```
$ oc get is
NAME      IMAGE REPOSITORY                                                   TAGS     UPDATED
patroni   image-registry.openshift-image-registry.svc:5000/patroni/patroni   latest   50 seconds ago
```

## Logging and populating database
To get the service IPs of the PostgreSQL cluster, run:

```
$ oc get svc
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
patroni-persistent           ClusterIP   172.30.88.252   <none>        5432/TCP   20h
patroni-persistent-master    ClusterIP   172.30.51.69    <none>        5432/TCP   20h
patroni-persistent-replica   ClusterIP   172.30.0.120    <none>        5432/TCP   20h
```

<b>Note:</b> You can use the patroni-persistent-master service to establish a read/write connection 
while the patroni-persistent-replica service can be used to establish a read only connection.

We are logging into PostgreSQL from the pgbench pod. The password for logging in is `postgres`.
```
oc rsh pgbench
psql -U postgres -h 172.30.51.69
```

To create a table and populate data in it, run the following
```
postgres=> CREATE TABLE TEMP(id INTEGER PRIMARY KEY, name VARCHAR(10));
postgres=> INSERT INTO TEMP VALUES(1,'alex');
```

To check if the data was populated do
```
postgres=> SELECT * FROM TEMP;
```
The output of the table should look like this
```
 id | name 
----+------
  1 | alex

```

## Quiescing the database

Velero hooks enable the execution of terminal commands before and after resources are backed up. 
Before a backup, "pre" hooks are used to freeze resources, so that they are not modified as the backup is taking place. 
After a backup, "post" hooks are used to unfreeze those resources so so they can accept new transactions.


- The Velero hooks are specified as annotations on the resource. The command to annotate it using oc is:
```
oc annotate pod -n patroni \
    pre.hook.backup.velero.io/command= '["/bin/bash", "-c","patronictl pause && pg_ctl stop -D pgdata/pgroot/data"]' \
    pre.hook.backup.velero.io/container=patroni-persistent \
    post.hook.backup.velero.io/command='["/bin/bash", "-c", "patronictl resume"]'\
    post.hook.backup.velero.io/container=patroni-persistent
```

These lines specify the "pre" hook to freeze resources:

The container specifies where the command should be executed. Patronictl pause stops down the patroni cluster and pg_ctl stop shuts down the server running in the specified data directory.
```
pre.hook.backup.velero.io/command: '["/bin/bash", "-c","patronictl pause && pg_ctl stop -D pgdata/pgroot/data"]'
pre.hook.backup.velero.io/container: patroni-persistent
```

These lines specify the "post" hook to unfreeze them:

The container specifies where the command should be executed. Patronictl resume starts up the patroni cluster.
```
post.hook.backup.velero.io/command: '["/bin/bash", "-c", "patronictl resume"]'
post.hook.backup.velero.io/container: patroni-persistent
```

## Back up the application
```
oc create -f postgres-backup.yaml 
```

To check if the CSI snapshots are created after backup, run the following command:
```
$ oc get volumesnapshotcontent
NAME                                                              READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER                       VOLUMESNAPSHOTCLASS       VOLUMESNAPSHOT                                         AGE
snapcontent-24ef293e-68b1-4f01-8d9b-673d20c6b423                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-0-6l8rf   46m
snapcontent-bea37537-a585-4c5d-a02d-ce1067f067a8                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-2-hk4pk   45m
snapcontent-c0e5a49b-e5d8-4235-8f90-96f2eedf2d04                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-1-q7nj6   46m
snapcontent-d1104635-17d9-4c83-82e4-94032b31054e                  true         2147483648    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-8gx2g                                   44m
```

## Delete the application.
Make sure the backup is completed (`oc get backup -n velero patroni -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
oc delete namespace patroni
```

## Restore the application.
```
oc create -f postgres-restore.yaml 
```
To check if the CSI snapshots are created after restore, run the following command:
```
$ oc get volumesnapshotcontent
NAME                                                              READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER                       VOLUMESNAPSHOTCLASS       VOLUMESNAPSHOT                                         AGE
snapcontent-24ef293e-68b1-4f01-8d9b-673d20c6b423                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-0-6l8rf   46m
snapcontent-bea37537-a585-4c5d-a02d-ce1067f067a8                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-2-hk4pk   45m
snapcontent-c0e5a49b-e5d8-4235-8f90-96f2eedf2d04                  true         5368709120    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-1-q7nj6   46m
snapcontent-d1104635-17d9-4c83-82e4-94032b31054e                  true         2147483648    Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-8gx2g                                   44m
velero-velero-patroni-8gx2g-hhgd2                                 true         0             Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-8gx2g                                   40m
velero-velero-patroni-persistent-patroni-persistent-0-6l8rcppgv   true         0             Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-0-6l8rf   17m
velero-velero-patroni-persistent-patroni-persistent-1-q7njpv44s   true         0             Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-1-q7nj6   17m
velero-velero-patroni-persistent-patroni-persistent-2-hk4pv8mgz   true         0             Retain           rook-ceph.rbd.csi.ceph.com   csi-rbdplugin-snapclass   velero-patroni-persistent-patroni-persistent-2-hk4pk   17m
```


