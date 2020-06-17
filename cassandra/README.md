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
This allows us to have access to the first node in the cassandra cluster. From here we can create a sample
keyspace and populate it.
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
## Back up the application.
```
ansible-playbook backup.yaml
```
## Delete application.
Make sure the backup is completed (`oc get backup -n <velero> cassandra -o jsonpath='{.status.phase}'`
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
```
oc exec -it cassandra-0 -- cqlsh
SELECT * FROM classicmodels.offices;
```