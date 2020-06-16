# Stateful application backup/restore (PostgreSQL)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. Restic is
used for PV backup. This setup assumes you have KUBECONFIG properly configured.

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


## Back up the application
```
ansible-playbook postgres-backup.yaml -e velero_namespace=<VELERO_NAMESPACE>
```
If the backup fails, try to check your aws bucket location by
```
oc project velero
oc edit volumesnapshotlocation
```
and change from us-east-2 to us-east-1 and vice versa based on your configuration of openshift installation bucket

=======

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

