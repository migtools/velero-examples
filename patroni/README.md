# Patroni
Patroni is a template for you to create your own customized, high-availability solution using Python and - for maximum accessibility - a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful.

# Installation
## Create project

```
oc new-project patroni
```

## Create the Template
Install the template into the `openshift` namespace if this should be shared across projects: 

```
oc create -f template_patroni_persistent.yaml
```
## Create a new app

```
oc new-app patroni-pgsql-persistent 
```
## Create pgbench pod
```
oc create -f pgbench.yaml
```
## Save state metadata prior to backup/restore.
```
oc get all -n patroni > patroni-running-before.txt
oc get pvc -n patroni >> patroni-running-before.txt
oc get pv >> patroni-running-before.txt
```
## For logging into the postgres database
For host value to pass to psql, use the CLUSTER-IP of the service. To get that do

```
oc get svc

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
patroni-persistent           ClusterIP   172.30.88.252   <none>        5432/TCP   20h
patroni-persistent-master    ClusterIP   172.30.51.69    <none>        5432/TCP   20h
patroni-persistent-replica   ClusterIP   172.30.0.120    <none>        5432/TCP   20h

```
For logging in

user/pass=postgres/postgres
```
oc rsh pgbench
psql -U postgres -h 172.30.51.69
```
Note: using master IP opens a connection with read/write access, using replica IP opens a connection with read access.

## Populate database
Login using the above commands to the database.
To create a table and populate data in it, run the following
```
postgres=> CREATE TABLE TEMP(id INTEGER PRIMARY KEY, name VARCHAR(10));
postgres=> INSERT INTO TEMP VALUES(1,'alex');
```
To check if the data was populated and table created, do
```
postgres=> SELECT * FROM TEMP;
postgres=> \dt
```
The output of the table should be like this
```
 id | name 
----+------
  1 | alex

```
## Back up the application
```
oc create -f postgres-backup.yaml 
```
If the backup fails, try to check your aws bucket location by
```
oc project <VELERO_NAMESPACE>
oc edit volumesnapshotlocation
```
and change your region based on your configuration of openshift installation bucket

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

## Compare application state before/after.
Make sure the restore is completed (`oc get restore -n velero patroni -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
oc get all -n patroni-persistent > patroni-running-after.txt
oc get pvc -n patroni-persistent >> patroni-running-after.txt
oc get pv >> patroni-running-after.txt

```
Compare "patroni-running-before.txt" and "patroni-running-after.txt"


# Quiescing the database

