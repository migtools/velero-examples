# Stateful backup/restore (cassandra)
Assumes storageclass of "gp2". Modify appropriately if needed.

Uses ansible roles for each of the following functionality below.

Execute each in velero-examples/cassandra.
## Create cassandra statefulset.
```
ansible-playbook install.yaml
```
## Back up the application.
```
ansible-playbook backup.yaml
```
## Delete application.
Make sure the backup is completed (`oc get backup -n velero cassandra -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
ansible-playbook delete.yaml
```

## Restore the application.
```
ansible-playbook restore.yaml
```
