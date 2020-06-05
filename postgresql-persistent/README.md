# Stateful application backup/restore (PostgreSQL)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. Restic is
used for PV backup.

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


## Back up the application
```
ansible-playbook postgres-backup.yaml
```
## Delete the application.
Make sure the backup is completed (`oc get backup -n velero postgres-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
ansible-playbook postgres-delete.yaml
```

## Restore the application.
```
ansible-playbook postgres-restore.yaml
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

