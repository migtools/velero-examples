# Stateful backup/restore (cassandra)
Assumes storageclass of "gp2". Modify appropriately if needed.

Assumes the use of velero in the oadp-operator and creates cassandra
in the namespace cassandra-stateful. Change Variables if needed to adjust.

Uses ansible roles for each of the following functionality below.

Execute each in velero-examples/cassandra.
## Create cassandra statefulset.
```
ansible-playbook install.yaml
```
## Populate a sample database
After the cassandra cluster is up, the following below
will access the first node and populate a sample database to test that
data persists during the backup and restore
process.
```
oc exec -it cassandra-0 -- cqlsh
```
This allows us to have access to the cassandra cluster. From here we can create a sample
keyspace and a table then we can populate the table in it with some sample data.
```
CREATE KEYSPACE classicmodels WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };
use classicmodels;
CREATE TABLE offices (officeCode text PRIMARY KEY, city text, phone text, addressLine1 text, addressLine2 text, state text, country text, postalCode text, territory text);
INSERT into offices(officeCode, city, phone, addressLine1, addressLine2, state, country ,postalCode, territory) values
('1','San Francisco','+1 650 219 4782','100 Market Street','Suite 300','CA','USA','94080','NA');
```
Verify that the data was populated.
```
SELECT * FROM classicmodels.offices;
```
Output should be the following
<pre> <font color="#CC0000"><b>officecode</b></font> | <font color="#75507B"><b>addressline1</b></font>      | <font color="#75507B"><b>addressline2</b></font> | <font color="#75507B"><b>city</b></font>          | <font color="#75507B"><b>country</b></font> | <font color="#75507B"><b>phone</b></font>           | <font color="#75507B"><b>postalcode</b></font> | <font color="#75507B"><b>state</b></font> | <font color="#75507B"><b>territory</b></font>
------------+-------------------+--------------+---------------+---------+-----------------+------------+-------+-----------
          <font color="#C4A000"><b>1</b></font> | <font color="#C4A000"><b>100 Market Street</b></font> |    <font color="#C4A000"><b>Suite 300</b></font> | <font color="#C4A000"><b>San Francisco</b></font> |     <font color="#C4A000"><b>USA</b></font> | <font color="#C4A000"><b>+1 650 219 4782</b></font> |      <font color="#C4A000"><b>94080</b></font> |    <font color="#C4A000"><b>CA</b></font> |        <font color="#C4A000"><b>NA</b></font>
</pre>
We can now exit the shell by simply exiting.
```
exit
```
For the quiesce behavior in cassandra, we will be using Cassandra Nodetool. Nodetool provides
a variety of utility to manage the cluster and perform database operations. Below will show how nodetool will be 
used during the backup.

Backup will look like the following with the nodetool operations being called.
* Disable Gossip (Stops communication with other nodes)
* Disable Thrift (Stops communication with one of the two protocols for listening to client)
* Disable Binary (Stops communication with the other protocol for listening to client)
* Nodetool flush is called to flush all memory to disk
* Perform Backup
* Enable Gossip, Thrift, and Binary (Start listening to connections again)

Note we have used velero hooks that preforms all this during the backup. They will look like this.
Prehooks are commands executed before backup and posthooks are ones after backup.
<pre>
pre.hook.backup.velero.io/command: '["/bin/bash", "-c", "opt/cassandra/bin/nodetool disablegossip && opt/cassandra/bin/nodetool disablethrift && opt/cassandra/bin/nodetool disablebinary && opt/cassandra/bin/nodetool flush"]'
post.hook.backup.velero.io/command: '["/bin/bash", "-c", "opt/cassandra/bin/nodetool enablegossip && opt/cassandra/bin/nodetool enablethrift && opt/cassandra/bin/nodetool enablebinary"]'
</pre>
## Back up the application.
```
ansible-playbook backup.yaml
```
We can see that the data is now in our sstables.
```
oc exec -it cassandra-0 nodetool getsstables classicmodels offices 1
```
A similar output should look like this
`/var/lib/cassandra/data/classicmodels/offices-1bb77060b65a11eaa47369447437c0db/md-1-big-Data.db`

## Delete application.
Make sure the backup is completed  
`oc get backup -n <velero> cassandra -o jsonpath='{.status.phase}'`
should show "Completed".
---
Also note that "velero" should be replaced with whatever namespace velero is in.  
Then, run:
```
ansible-playbook delete.yaml
```

## Restore the application.
```
ansible-playbook restore.yaml
```
## Check that the data persists afer doing a backup and restore
Note that connecting to the node after the restore will take some time.
At this point there are a few things we can check to make sure data is not only in our keyspace but also
still in our sstables.
```
oc exec -it cassandra-0 -- cqlsh
SELECT * FROM classicmodels.offices;
```
Output should be the same as the first time it was executed.
Now exit the shell to make sure that the data is still in sstables.
```
exit
```
Lets run nodetool getsstables to check the .db file still exists and
nodetool getendpoints to make sure that the data was replicated across all nodes.
```
oc exec -it cassandra-0 -- nodetool getsstables classicmodels offices 1
```
This should output the same file as the first file after cassandra was drained.
Next we can check that the data is replicated across the nodes.
```
oc exec -it cassandra-0 nodetool getendpoints classicmodels offices 1
```
This will give something like this

10.128.3.237

10.131.0.92

10.129.2.191
