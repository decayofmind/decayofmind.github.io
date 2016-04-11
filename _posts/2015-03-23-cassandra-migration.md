---
title: "Migrating data between Apache Cassandra clusters"
layout: post
date: 2015-03-23 12:00:00
tag:
- cassandra
- migration
blog: true
---

This was originally posted by me on [Glogster dev blog](https://glogster.github.io/posts/2015/03/23/cassandra-migration.html).

A couple of weeks ago, we suspended our social network service for teens, which was based on Apache Cassandra, and we aim to keep that service in read-only state for as long as we can. In the mean time, our priority is moving forward with our EDU product, which uses Cassandra as storage for raw data. Those two keyspaces will become incompatible soon, in terms of the partitioners and data models used, resulting in different performance issues and bottlenecks. To make the retired glogster.com site independent from the main infrastructure, its data (keyspace) needs to be migrated to another Cassandra cluster.

Here I want to describe how I have solved this problem, as I've not found any detailed information about migrations from cluster to cluster in Cassandra.

The whole process contains following steps:

1. Dumping of the current schema
2. Copying the new schema to the new cluster node
4. Making a snapshot (superset) of the target keyspace of the old cluster
5. Transferring those snapshots to the new cluster node
6. Running repair and cleanup on the new cluster node

To begin with, lets discuss dumping the current schema:
cqlsh utility in 1.2 doesn't support "-e" option anymore, so we may only execute commands provided in file. Lets create a file containing this:

```
echo "DESC KEYSPACE MyKeySpace;" > cqlcmdf
```

and then pass it to cqlsh:

```
cqlsh -f cqlcmdf | tail -n +2 > my_schema.cdl
```

Then, edit my_schema.cdl with the proper replication factor.

Assume that we have a clean and configured installation of Cassandra at the new cluster and that the cdl file with our schema is already transferred to that machine. Start the Cassandra service and run:

```
cqlsh -f my_schema.cdl
```

Be careful, if your original tables were created with cassandra-cli (old Cassandra versions) you'll need to migrate your schema with cassandra-cli too.

```
echo -e "use MyKeySpace;\r\n show schema;\n" | cassandra-cli -h localhost > mySchema.cdl
```

and then

```
cassandra-cli -h localhost -f mySchema.cdl
```

Next, we need to create a snapshot of the target keyspace (make sure you have JNA enabled on your node):

```
nodetool snapshot MyKeySpace snap_1
```

where *MyKeySpace* is the name of the target keyspace and *snap_1* is the snapshot name.

Once this is complete, the snapshot will populate the *snapshot* dir of every collumnfamily dir.

In our particular case, our RF on old cluster was equal to the total number of nodes and the target cluster contains only one node, so after copying all of the sstables to it, we only needed to run nodetool refresh on all tables in our keyspace:

```
cd <cassandra_data_dir>
for i in `ls MyKeySpace/`;do nodetool refresh MyKeySpace $i; done
```

After the data is successfully loaded (this may take a significant amount of time) we need to run a nodetool cleanup on that keyspace, so:

```
nodetool cleanup
```

This will also take some time. Finally, here we have a working cluster with migrated data.
