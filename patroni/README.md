# Patroni
Patroni is a template for you to create your own customized, high-availability solution using Python and - for maximum accessibility - a distributed configuration store like ZooKeeper, etcd, Consul or Kubernetes. Database engineers, DBAs, DevOps engineers, and SREs who are looking to quickly deploy HA PostgreSQL in the datacenter-or anywhere else-will hopefully find it useful.

# Installation
## Create test project

```
oc new-project patroni
```
## Build the image

Note: If deploying as a template for multiple users, the following commands should be performed in a shared namespace like `openshift`. 

```
docker pull jaydipgabani/postgres
```
## Create the Template
Install the template into the `openshift` namespace if this should be shared across projects: 

```
oc create -f template_patroni_persistent.yaml -n openshift
```
## Create a new app

```
oc new-app patroni-pgsql-persistent 
```

# Quiescing the database

