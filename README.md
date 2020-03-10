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

[Stateless example](nginx-stateless/README.md)

## Stateful application backup/restore (mysql)

Persistent application, backup/restore using Restic.

[Stateful example](mysql-persistent/README.md)

## Application backup/restore with NooBaa

[Stateful example with NooBaa](noobaa-example/README.md)

## Stateful application backup/restore using snapshots (mssql)

Persistent application, backup/restore using PV snapshots.

[Stateful example](mssql-persistent/README.md)
