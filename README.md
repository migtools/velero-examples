# velero-examples

This repo provides a few examples using velero to back up and restore
applications. The CLI invocations use kubectl where it would work, and
oc when there are openshift-specific resources involved. In a base
kubernetes environment, some modification might be needed (use
Deployments instead of DeploymentConfigs, etc.)

## Install Velero (1.2 release)

The velero image used is the upstream 1.2 release from docker.io. The
only plugin included is the aws plugin required for supporting an
aws/s3 BackupStorageLocation

TODO: modify scripts to take VELERO_* from env/input to avoid risk of committing credentials to github.

- replace VELERO_ACCESS_KEY_ID and VELERO_SECRET_ACCESS_KEY in velero-install/aws-credentials with your values (make sure this change doesn't get committed to github)
- replace VELERO_BUCKET_NAME in velero-install/velero-server.yaml with your bucket name

```
kubectl apply -f velero-install/crds.yaml
kubectl apply -f velero-install/velero-namespace.yaml
kubectl create secret generic cloud-credentials --namespace velero --from-file cloud=velero-install/aws-credentials
kubectl apply -f velero-install/velero-server.yaml
```

## Stateless application backup/restore (nginx)

### Create stateless nginx deployment.
```
oc create -f nginx-stateless/nginx-deployment.yaml #uses openshift-specific Route
kubectl get all -n nginx-example > nginx-running-before.txt
```
### Back up the application.
```
kubectl create -f nginx-stateless/nginx-stateless-backup.yaml
```
### Delete application.
Make sure the backup is completed (`kubectl get backup -n velero nginx-stateless -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace nginx-example
```

### Restore the application.
```
kubectl create -f nginx-stateless/nginx-stateless-restore.yaml
```

### Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero nginx-stateless -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n nginx-example > nginx-running-after.txt
```
Compare "nginx-running-before.txt" and "nginx-running-after.txt"

## Stateful application backup/restore (mysql)

The persistent case assumes the existence of the storageclass "gp2" --
modify appropriately if this is incorrect for your cluster.

### Create stateful mysql deploymentconfig:
```
oc create -f mysql-persistent/mysql-persistent-template.yaml
```

Now add an empty file to the PV to assist in confirming a successful
restore:
```
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod touch /var/lib/mysql/data/foo; done
```

### Save state metadata prior to backup/restore.
```
kubectl get all -n mysql-persistent > mysql-running-before.txt
kubectl get pvc -n mysql-persistent >> mysql-running-before.txt
kubectl get pv >> mysql-running-before.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-running-before.txt
```

### Back up the application
```
kubectl create -f mysql-persistent/mysql-persistent-backup.yaml
```
### Delete the application.
Make sure the backup is completed (`kubectl get backup -n velero mysql-persistent -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace mysql-persistent
```
### Restore the application.
```
kubectl create -f mysql-persistent/mysql-persistent-restore.yaml
```

### Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero mysql-persistent -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n mysql-persistent > mysql-running-after.txt
kubectl get pvc -n mysql-persistent >> mysql-running-after.txt
kubectl get pv >> mysql-running-after.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-running-after.txt
```
Compare "mysql-persistent-before.txt" and "mysql-persistent-after.txt"
