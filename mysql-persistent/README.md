# Stateful application backup/restore (mysql)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. Restic is
used for PV backup.

## Create stateful mysql deploymentconfig:
```
oc create -f mysql-persistent/mysql-persistent-template.yaml
```

Now add an empty file to the PV to assist in confirming a successful
restore:
```
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod touch /var/lib/mysql/data/foo; done
```

## Save state metadata prior to backup/restore.
```
kubectl get all -n mysql-persistent > mysql-running-before.txt
kubectl get pvc -n mysql-persistent >> mysql-running-before.txt
kubectl get pv >> mysql-running-before.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-running-before.txt
```

## Back up the application
```
kubectl create -f mysql-persistent/mysql-persistent-backup.yaml
```
## Delete the application.
Make sure the backup is completed (`kubectl get backup -n velero mysql-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace mysql-persistent
```
## Restore the application.
```
kubectl create -f mysql-persistent/mysql-persistent-restore.yaml
```

## Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero mysql-persistent -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n mysql-persistent > mysql-running-after.txt
kubectl get pvc -n mysql-persistent >> mysql-running-after.txt
kubectl get pv >> mysql-running-after.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-running-after.txt
```
Compare "mysql-running-before.txt" and "mysql-running-after.txt"
