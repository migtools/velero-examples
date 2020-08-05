# Stateful application backup/restore (PostgreSQL)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. This setup assumes you have KUBECONFIG properly configured.

To use restic for PV backup, add the annotations in deployment.yaml
```
 template:
  metadata:
    annotations:
       backup.velero.io/backup-volumes: postgresql-data
```


## For installing missing modules
(for python3)
```
pip3 install kubernetes
pip3 install openshift
```

## Create stateful PostgreSQL deployment:
```
ansible-playbook postgres-install.yaml
```
Switch to postgresql-persistent namespace.
```
oc project postgresql-persistent
```

## Save state metadata prior to backup/restore.
```
oc get all -n postgresql-persistent > postgresql-running-before.txt
oc get pvc -n postgresql-persistent >> postgresql-running-before.txt
oc get pv >> postgresql-running-before.txt
```

## For installing/deleting specific resources(Pods, Service, Deployment etc..)
Run the following command by replacing tags with suitable tag name as mentioned in yaml
```
ansible-playbook postgres-install.yaml --tags "tagname(s)"
ansible-playbook postgres-delete.yaml --tags "tagname(s)"

```
## For logging into the postgres database
For host value to pass to psql, use the CLUSTER-IP of the service. To get that do

```
oc get svc
```
For logging in
user/pass=admin/password
```
oc rsh pgbench
oc exec -it pgbench bash
psql -U admin -W sampledb -h 172.30.87.66
```

## Populate a sample database
Login using the above commands to the database.
To create a table and populate data in it, run the following
```
sampledb=> CREATE TABLE TEMP(id INTEGER PRIMARY KEY, name VARCHAR(10));
sampledb=> INSERT INTO TEMP VALUES(1,'alex');
```
To check if the data was populated and table created, do
```
sampledb=> SELECT * FROM TEMP;
sampledb=> \dt
```
The output of the table should be like this
```
 id | name 
----+------
  1 | alex

```
## Back up the application
```
ansible-playbook postgres-backup.yaml -e velero_namespace=<VELERO_NAMESPACE>
```
If the backup fails, try to check your aws bucket location by
```
oc project <VELERO_NAMESPACE>
oc edit volumesnapshotlocation
```
and change your region based on your configuration of openshift installation bucket



## Delete the application.
Make sure the backup is completed (`oc get backup -n velero postgres-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
ansible-playbook postgres-delete.yaml
```

## Restore the application.
```
ansible-playbook postgres-restore.yaml -e velero_namespace=<VELERO_NAMESPACE>
```

## Compare application state before/after.
Make sure the restore is completed (`oc get restore -n velero postgres-persistent -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
oc get all -n postgresql-persistent > postgresql-running-after.txt
oc get pvc -n postgresql-persistent >> postgresql-running-after.txt
oc get pv >> postgresql-running-after.txt

```
Compare "postgresql-running-before.txt" and "postgresql-running-after.txt"

