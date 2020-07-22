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
<<<<<<< HEAD
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
At this stage in the backup and restore, we want to ensure that the data we just created will be consistent and reliable.
In order to achieve that, we need to able to quiesce cassandra. We can achieve this in cassandra by using a cassandra tool called
nodetool. This will allow us to flush memory onto what are called sstables that store the data in immutable files. 
Nodetool also has features that verifies the files were flushed and replicated across the nodes.

To start the quiesce feature in cassandra, we need to run the command Drain that flushes all memory from memtables in cassandra into
immutable sstables. Drain will also stop listening for connections from the client and other nodes.
nodetool. Using this gives access to many features and commands. One of them called drain will allow us to flush memory onto what are called sstables that store the data in immutable files. 
Nodetool will also be used to verifiy the data were flushed to files and replicated across the nodes at the end.

Verify first that the data isn't in our sstabless yet. There should be no output printed after running nodetool getsstables.
Once that happens, then drain the data to them.

```
oc exec -it cassandra-0 -- nodetool getsstables classicmodels offices 1
oc exec -it cassandra-0 -- nodetool drain
```
Now before we take a backup of cassandra, lets also verify that cassandra has stop
listening for connections. Try to access the cqlsh again. The following command should be an 
From here we could manually quiesce cassandra by doing (`oc exec -it cassandra-0 -- nodetool drain` but when we do our backup, we have used velero hooks to do that instead pre-backup.
## Back up the application.
```
ansible-playbook backup.yaml
```
After taking the backup of cassandra, lets also verify that cassandra has stop
listening for connections. This means writes have also stopped occurring temporarily. Try to access the cqlsh again. The following command should be an 
error like the following below after executing the command.
```
oc exec -it cassandra-0 -- cqlsh
```
Connection error: ('Unable to connect to any servers', {'127.0.0.1': error(111, "Tried connecting to [('127.0.0.1', 9042)]. Last error: Connection refused")})
command terminated with exit code 1.

Last thing before the backup, lets see that sstables have our data in .db files.
After the backup, lets see that sstables are now in our sstables.
```
oc exec -it cassandra-0 nodetool getsstables classicmodels offices 1
```
A similar output should look like this
/var/lib/cassandra/data/classicmodels/offices-1bb77060b65a11eaa47369447437c0db/md-1-big-Data.db
## Back up the application.
```
ansible-playbook backup.yaml
```

## Delete application.Make sure the backup is completed (`oc get backup -n <velero> cassandra -o jsonpath='{.status.phase}'`
should show "Completed"). Then, run:
---
Note that "velero" should be replaced with whatever namespace velero is in.
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
