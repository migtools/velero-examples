# Stateless application backup/restore (nginx)

## Create stateless nginx deployment.
```
oc create -f nginx-stateless/nginx-deployment.yaml #uses openshift-specific Route
kubectl get all -n nginx-example > nginx-running-before.txt
```
## Back up the application.
```
kubectl create -f nginx-stateless/nginx-stateless-backup.yaml
```
## Delete application.
Make sure the backup is completed (`kubectl get backup -n velero nginx-stateless -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace nginx-example
```

## Restore the application.
```
kubectl create -f nginx-stateless/nginx-stateless-restore.yaml
```

## Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero nginx-stateless -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n nginx-example > nginx-running-after.txt
```
Compare "nginx-running-before.txt" and "nginx-running-after.txt"


