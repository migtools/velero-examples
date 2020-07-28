# Patroni
Patroni is a template for you to create your own customized, high-availability solution using Python and for maximum accessibility 
a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. 
Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly 
deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful.

## Installation
The setup assumes you have installed velero in oadp-operator namespace. If velero is installed in other namespace, 
change the `namespace` field in `postgres-backup.yaml` and `postgres-restore.yaml` files.

To install patroni run the following commands,

```
oc new-project patroni
oc new-build . -n patroni --name=patroni
oc create -f template_patroni_persistent.yaml -n openshift
oc new-app patroni-pgsql-persistent 
oc create -f pgbench.yaml
```

The command `oc new-build . -n patroni --name=patroni` pushes the image to internal image registry. To check that, run the following commands,

```
$ oc get istag
NAME             IMAGE REFERENCE                                                                                                                            UPDATED
patroni:latest   image-registry.openshift-image-registry.svc:5000/patroni/patroni@sha256:d879c4f6502cc48b69d4aceefa4ff166b2900ff8d11b30937c59da20e3711aa5   44 seconds ago
$ oc get is
NAME      IMAGE REPOSITORY                                                   TAGS     UPDATED
patroni   image-registry.openshift-image-registry.svc:5000/patroni/patroni   latest   50 seconds ago
```

## Logging and populating database
To get the service IPs of the PostgreSQL cluster, run:

```
oc get svc
```
```
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


