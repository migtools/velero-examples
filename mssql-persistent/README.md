# Stateful application backup/restore using snapshots (mssql)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster. Snapshots
are used for PV backup.

## Restore velero AWS credentials secret

If you've configured velero for noobaa, the AWS cloud-credentials
secret has been overwritten with the noobaa credentials. The AWS
credentials need to be restored.
```
kubectl create secret generic cloud-credentials --namespace velero --from-file cloud=velero-install/aws-credentials --dry-run -o yaml --save-config|kubectl apply -f -
```

## Create stateful mysql deploymentconfig:
```
oc create -f mssql-persistent/mssql-persistent-template.yaml
```

Now add an empty file to the PV to assist in confirming a successful
restore:
```
for pod in $(oc get pod -n mssql-persistent --field-selector=status.phase=Running --selector=app=mssql --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mssql-persistent $pod touch /var/opt/mssql/data/foo; done
```

## Save state metadata prior to backup/restore.
```
kubectl get all -n mssql-persistent > mssql-running-before.txt
kubectl get pvc -n mssql-persistent >> mssql-running-before.txt
kubectl get pv >> mssql-running-before.txt
for pod in $(oc get pod -n mssql-persistent --field-selector=status.phase=Running --selector=app=mssql --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mssql-persistent $pod ls -hal /var/opt/mssql/data/foo; done >> mssql-running-before.txt
```

## Back up the application
```
kubectl create -f mssql-persistent/mssql-persistent-backup.yaml
```
## Delete the application.
Make sure the backup is completed (`kubectl get backup -n velero mssql-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace mssql-persistent
```

## Restore the application.
```
kubectl create -f mssql-persistent/mssql-persistent-restore.yaml
```

## Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero mssql-persistent -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n mssql-persistent > mssql-running-after.txt
kubectl get pvc -n mssql-persistent >> mssql-running-after.txt
kubectl get pv >> mssql-running-after.txt
for pod in $(oc get pod -n mssql-persistent --field-selector=status.phase=Running --selector=app=mssql --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mssql-persistent $pod ls -hal /var/opt/mssql/data/foo; done >> mssql-running-after.txt
```
Compare "mssql-running-before.txt" and "mssql-running-after.txt"
