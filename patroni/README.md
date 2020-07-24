# Patroni
Patroni is a template for you to create your own customized, high-availability solution using Python and - for maximum accessibility - a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful.

## Installation

<b>Note:</b> Check your AWS VSL location and change your region based on your configuration if the velero backup fails.

To install patroni run the following commands,

```
oc new-project patroni
oc create -f template_patroni_persistent.yaml -n openshift
oc new-app patroni-pgsql-persistent 
oc create -f pgbench.yaml
```

## Logging and populating database
For host value to pass to psql, use the CLUSTER-IP of the master pod. To get that do

```
oc get svc
```
```
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
patroni-persistent           ClusterIP   172.30.88.252   <none>        5432/TCP   20h
patroni-persistent-master    ClusterIP   172.30.51.69    <none>        5432/TCP   20h
patroni-persistent-replica   ClusterIP   172.30.0.120    <none>        5432/TCP   20h
```

<b>Note:</b> Master IP opens a connection with read/write access while replica IP opens a connection with read access.



Populate the database by logging into pgbench pod. The password for logging in is `postgres`.
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

Velero hooks enable the execution of terminal commands before and after resources are backed up. Before a backup, "pre" hooks are used to freeze resources, so that they are not modified as the backup is taking place. After a backup, "post" hooks are used to unfreeze those resources so that can altered again.


- The Velero hooks are specified as annotations in the template.

These lines specify the "pre" hook to freeze resources:

```
pre.hook.backup.velero.io/command: '["/bin/bash", "-c","patronictl pause && pg_ctl stop -D pgdata/pgroot/data"]'
pre.hook.backup.velero.io/container: patroni-persistent
```

These lines specify the "post" hook to unfreeze them:

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


