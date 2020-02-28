# Application backup/restore with NooBaa

This case uses the same application definition from the Mysql
persistent example, but velero is configured to backup and restore
using NooBaa instead of AWS.

## Install Noobaa

```
oc apply -f noobaa-example/namespace.yaml
wget https://github.com/noobaa/noobaa-operator/releases/download/v2.0.9/noobaa-linux-v2.0.9; mv noobaa-linux-* noobaa; chmod +x noobaa
./noobaa install -n noobaa
```
Wait for completion.
Saved output.
```
oc apply -f noobaa-example/pv-pool.yaml

export NOOBAA_ENDPOINT=`oc get service -n noobaa s3 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'`
export NOOBAA_S3_ACCESS_KEY_ID=`oc get secret -n noobaa noobaa-admin -o jsonpath='{.data.AWS_ACCESS_KEY_ID}'|base64 -d`
export NOOBAA_S3_SECRET_ACCESS_KEY=`oc get secret -n noobaa noobaa-admin -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}'|base64 -d`

cat >noobaa-credentials <<EOF
[default]
aws_access_key_id=$NOOBAA_S3_ACCESS_KEY_ID
aws_secret_access_key=$NOOBAA_S3_SECRET_ACCESS_KEY
EOF
kubectl create secret generic cloud-credentials --namespace velero --from-file cloud=noobaa-credentials --dry-run -o yaml --save-config|kubectl apply -f -

cat <<EOF | kubectl create -f -
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  creationTimestamp: null
  labels:
    component: velero
  name: noobaa
  namespace: velero
spec:
  config:
    region: "us-east-2"
    s3ForcePathStyle: "true"
    insecureSkipTLSVerify: "true"
    s3Url: http://$NOOBAA_ENDPOINT
  objectStorage:
    bucket: "velero-examples-storage"
    prefix: "velero"
  provider: aws
EOF

```

## Create stateful mysql deploymentconfig (not necessary if this
already exists from prior restore):
```
oc create -f mysql-persistent/mysql-persistent-template.yaml
```

Now add an empty file to the PV to assist in confirming a successful
restore (not necessary if this already exists from prior restore):
```
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod touch /var/lib/mysql/data/foo; done
```

## Save state metadata prior to backup/restore.
```
kubectl get all -n mysql-persistent > mysql-noobaa-running-before.txt
kubectl get pvc -n mysql-persistent >> mysql-noobaa-running-before.txt
kubectl get pv >> mysql-noobaa-running-before.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-noobaa-running-before.txt
```

## Back up the application
```
kubectl create -f noobaa-example/noobaa-backup.yaml
```
## Delete the application.
Make sure the backup is completed (`kubectl get backup -n velero mysql-persistent-noobaa -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
```
kubectl delete namespace mysql-persistent
```
## Restore the application.
```
kubectl create -f noobaa-example/noobaa-restore.yaml
```
## Compare application state before/after.
Make sure the restore is completed (`kubectl get restore -n velero mysql-persistent-noobaa -o jsonpath='{.status.phase}'`)
should show "Completed", and the application pod should be
running. Now run:
```
kubectl get all -n mysql-persistent > mysql-noobaa-running-after.txt
kubectl get pvc -n mysql-persistent >> mysql-noobaa-running-after.txt
kubectl get pv >> mysql-noobaa-running-after.txt
for pod in $(oc get pod -n mysql-persistent --field-selector=status.phase=Running --no-headers | awk '{print $1}'); do echo $pod; oc rsh -n mysql-persistent $pod ls -hal /var/lib/mysql/data/foo; done >> mysql-noobaa-running-after.txt
```
Compare "mysql-noobaa-running-before.txt" and "mysql-noobaa-running-after.txt"
