# Stateful backup/restore (cassandra)

## Create cassandra statefulset
```
ansible-playbook install.yaml
```
## Back up the application.
```
ansible-playbook backup.yaml
```
## Delete application.
```
ansible-playbook delete.yaml
```

## Restore the application.
```
ansible-playbook restore.yaml
```
