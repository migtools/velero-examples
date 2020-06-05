# Stateful application backup/restore (PostgreSQL)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. Restic is
used for PV backup.

## Create stateful PostgreSQL deploymentconfig:
```
ansible-playbook postgres-install.yaml
```

Now add an empty file to the PV to assist in confirming a successful
restore:
```
for pod in $(oc get pod -n postgresql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n postgresql-persistent $pod touch /var/lib/pgsql/data/foo; done
```

## Save state metadata prior to backup/restore.
```
oc get all -n postgresql-persistent > postgresql-running-before.txt
oc get pvc -n postgresql-persistent >> postgresql-running-before.txt
oc get pv >> postgresql-running-before.txt
for pod in $(oc get pod -n postgresql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n postgresql-persistent $pod ls -hal /var/lib/pgsql/data/foo; done >> postgresql-running-before.txt
```

## Back up the application
```
ansible-playbook postgres-backup.yaml
```
## Delete the application.
Make sure the backup is completed (`oc get backup -n velero postgresql-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
ansible-playbook postgres-delete.yaml
```
## Restore the application.
```
ansible-playbook postgres-restore.yaml
```

## Compare application state before/after.
Make sure the restore is completed (`oc get restore -n velero postgresql-persistent -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
oc get all -n postgresql-persistent > postgresql-running-after.txt
oc get pvc -n postgresql-persistent >> postgresql-running-after.txt
oc get pv >> postgresql-running-after.txt
for pod in $(oc get pod -n postgresql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n postgresql-persistent $pod ls -hal /var/lib/pgsql/data/foo; done >> postgresql-running-after.txt
```
Compare "postgresql-running-before.txt" and "postgresql-running-after.txt"

