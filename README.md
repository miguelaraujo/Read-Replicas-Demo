
Read Replicas
===============

General information and auxiliary diagrams available at: https://pt.slideshare.net/miguelgaraujo/deep-dive-into-mysql-innodb-cluster-read-scaleout-capabilitiespdf

Pre-requisites
--------------
  - MySQL Server 8.0.23+
  - MySQL Shell 8.1.0+
  - MySQL Router 8.1.0+

MySQL Instances
---------------
1) Deploy 18 MySQL Instances running on the following ports:

  - "Rome": [30001, 30006]
  - "Brussels": [30011, 30016]
  - "Lisbon": [30021, 30026]

2) Ensure the MySQL instances configuration is ready for InnoDB Cluster usage and create a cluster administration account

Example:

    mysqlsh-js> dba.configureInstance("root@rome:30001", {clusterAdmin: "clusteradmin"})

Demo
----

Create standalone cluster @ Lisbon
----------------------------------

    $ mysqlsh
    mysqlsh-js> \c clusteradmin@lisbon:30021
    mysqlsh-js> lis = dba.createCluster("Lisbon")
    mysqlsh-js> lis.addInstance("lisbon:30022")
    mysqlsh-js> lis.addInstance("rome:30023")

**Create a Router administration account**

    mysqlsh-js> rom.setupRouterAccount("routeradmin")


**Bootstrap and start the Router**

    $ mysqlrouter --bootstrap clusteradmin@lisbon:30021 -d router_rome --name="router_rome" --account=routeradmin --report-host=rome
    $ ./router_rome/start.sh

*NOTE*: Check router's log:

    $ tail -f router_rome/log/mysqlrouter.log

Start RW and RO applications
----------------------------
    $ python app_rw.py
    $ python app_ro.py

*NOTE*: Change the connection configuration to use the right host being tested

Create ClusterSet @ Rome and Brussels
-------------------------------------
    mysqlsh-js> \c clusteradmin@rome:30001
    mysqlsh-js> rom = dba.createCluster("Rome")
    mysqlsh-js> rom.addInstance("rome:30002")
    mysqlsh-js> rom.addInstance("rome:30003")
    mysqlsh-js> cs = rom.createClusterSet("clusterset")

**Create Replica Cluster @ Brussels:**

    mysqlsh-js> bru = cs.createReplicaCluster("clusteradmin@brussels:30011", "Brussels")
    mysqlsh-js> bru.addInstance("brussels:30012")
    mysqlsh-js> bru.addInstance("brussels:30013");

**Bootstrap Router @ Rome:**

    $ mysqlrouter --bootstrap clusteradmin@rome:30001 -d router_rome --name="router_rome" --account=routeradmin --report-host=rome
    $ ./router_rome/start.sh


Create Read Replica @ Lisbon
----------------------------
    mysqlsh-js> \c clusteradmin@lisbon:300021
    mysqlsh-js> c = dba.getCluster()
    mysqlsh-js> c.addReadReplica("lisbon:30024")

**Check topology info and status**

   mysqlsh-js> c.describe()
   mysqlsh-js> c.status()


Create Read Replica with different configuration @ Lisbon
---------------------------------------------------------
    mysqlsh-js> c.addReadReplica("lisbon:30025", {replicationSources: "secondary"})

Configure Router's destination pool
-----------------------------------
**Check current routing options**

    mysqlsh-js> c.routingOptions()

**Change destination pool configuration**

    mysqlsh-js> c.setRoutingOptions("read_only_targets", "all")

Hide a Read Replica
-------------------
    mysqlsh-js> c.setInstanceOption("lisbon:30025", "tag:_hidden", true)

**Observe Router's behavior and unhide the instance**

   mysqlsh-js> c.setInstanceOption("lisbon:30025", "tag:_hidden", null)

Change failover candidates
--------------------------
    mysqlsh-js> c.setInstanceOption("lisbon:30024", "replicationSources", "secondary")

**Apply change immediately**

    mysqlsh-js> c.rejoinInstance("lisbon:300024")

Trigger automatic failover
--------------------------
**Kill mysqld process of the secondary being used as source for the read-replicas**

   mysqlsh-js> c.status()

Add Read Replica to the ClusterSet
----------------------------------
**Create read replicas in Rome**

    mysqlsh-js> \c clusteradmin@rome:30001
    mysqlsh-js> c = dba.getCluster()
    mysqlsh-js> c.addReplicaInstance("rome:30004")
    mysqlsh-js> c.addReplicaInstance("rome:30005", {replicationSources: "secondary"})
    mysqlsh-js> c.addReplicaInstance("rome:30006", {replicationSources: ["rome:30001", "rome:30002"]})

**Create read-replicas in Brussels**

    mysqlsh-js> \c clusteradmin@brussels:30011
    mysqlsh-js> c = dba.getCluster()
    mysqlsh-js> c.addReplicaInstance("brussels:30014")
    mysqlsh-js> c.addReplicaInstance("brussels:30015")
    mysqlsh-js> c.addReplicaInstance("brussels:30016")

Configure Router
----------------
    mysqlsh-js> cs = dba.getClusterSet()
    mysqlsh-js> cs.routingOptions()
    mysqlsh-js> cs.setRoutingOption("read_only_targets", "read_replicas")
    mysqlsh-js> cs.setRoutingOption("rome::router_rome", "target_cluster", "brussels")

**Observe Router's behavior**

ClusterSet Primary Switchover
-----------------------------
    mysqlsh-js> cs.setPrimaryCluster("Brussels")

**Observe Router's behavior**

**Switchover back to Rome**

    mysqlsh-js> cs.setPrimaryCluster("Rome")


