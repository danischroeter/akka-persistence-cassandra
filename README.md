Cassandra Plugins for Akka Persistence
======================================

Replicated [Akka Persistence](http://doc.akka.io/docs/akka/2.3.6/scala/persistence.html) journal and snapshot store backed by [Apache Cassandra](http://cassandra.apache.org/).

[![Build Status](https://travis-ci.org/krasserm/akka-persistence-cassandra.svg?branch=master)](https://travis-ci.org/krasserm/akka-persistence-cassandra)

Dependency
----------

To include the Cassandra plugins into your `sbt` project, add the following lines to your `build.sbt` file:

    resolvers += "krasserm at bintray" at "http://dl.bintray.com/krasserm/maven"

    libraryDependencies += "com.github.krasserm" %% "akka-persistence-cassandra" % "0.3.4"

This version of `akka-persistence-cassandra` depends on Akka 2.3.6 and is cross-built against Scala 2.10.4 and 2.11.0. It is compatible with Cassandra 2.0.3 or higher. Versions of the Cassandra plugins that are compatible with Cassandra 1.2.x are maintained on the [cassandra-1.2](https://github.com/krasserm/akka-persistence-cassandra/tree/cassandra-1.2) branch.   

Journal plugin
--------------

### Features

- All operations required by the Akka Persistence [journal plugin API](http://doc.akka.io/docs/akka/2.3.6/scala/persistence.html#journal-plugin-api) are fully supported.
- The plugin uses Cassandra in a pure log-oriented way i.e. data are only ever inserted but never updated (deletions are made on user request only or by persistent channels, see also [Caveats](#caveats)).
- Writes of messages and confirmations are batched to optimize throughput. See [batch writes](http://doc.akka.io/docs/akka/2.3.6/scala/persistence.html#batch-writes) for details how to configure batch sizes. The plugin was tested to work properly under high load.
- Messages written by a single processor are partitioned across the cluster to achieve scalability with data volume by adding nodes.

### Configuration

To activate the journal plugin, add the following line to your Akka `application.conf`:

    akka.persistence.journal.plugin = "cassandra-journal"

This will run the journal with its default settings. The default settings can be changed with the following configuration keys:

- `cassandra-journal.contact-points`. A comma-separated list of contact points in a Cassandra cluster. Default value is `[127.0.0.1]`. Host:Port pairs are also supported. In that case the port parameter will be ignored.
- `cassandra-journal.port`. Port to use to connect to the Cassandra host. Default value is `9042`. Will be ignored if the contact point list is defined by host:port pairs.
- `cassandra-journal.keyspace`. Name of the keyspace to be used by the plugin. If the keyspace doesn't exist it is automatically created. Default value is `akka`.
- `cassandra-journal.table`. Name of the table to be used by the plugin. If the table doesn't exist it is automatically created. Default value is `messages`.
- `cassandra-journal.replication-factor`. Replication factor to use when a keyspace is created by the plugin. Default value is `1`.
- `cassandra-journal.max-partition-size`. Maximum number of entries (messages, confirmations and deletion markers) per partition. Default value is 5000000. **Do not change this setting after table creation** (not checked yet).
- `cassandra-journal.max-result-size`. Maximum number of entries returned per query. Queries are executed recursively, if needed, to achieve recovery goals. Default value is 50001.
- `cassandra-journal.write-consistency`. Write consistency level. Default value is `QUORUM`.
- `cassandra-journal.read-consistency`. Read consistency level. Default value is `QUORUM`.

The default read and write consistency levels ensure that processors can read their own writes. During normal operation, processors only write to the journal, reads occur only during recovery.

To connect to the Cassandra hosts with credentials, add the following lines:

- `cassandra-journal.authentication.username`. The username to use to login to Cassandra hosts. No authentication is set as default.
- `cassandra-journal.authentication.password`. The password corresponding to username. No authentication is set as default.

To limit the Cassandra hosts this plugin connects with to a specific datacenter, use the following setting:

- `cassandra-journal.local-datacenter`.  The id for the local datacenter of the Cassandra hosts that this module should connect to.  By default, this property is not set resulting in Datastax's standard round robin policy being used.

### Caveats

- Detailed tests under failure conditions are still missing.
- Range deletion performance (i.e. `deleteMessages` up to a specified sequence number) depends on the extend of previous deletions
    - linearly increases with the number of tombstones generated by previous permanent deletions and drops to a minimum after compaction
    - linearly increases with the number of plugin-level deletion markers generated by previous logical deletions (recommended: always use permanent range deletions)

These issues are likely to be resolved in future versions of the plugin.

Snapshot store plugin
---------------------

### Features

- Implements its own handler of the (internal) Akka Persistence snapshot protocol, making snapshot IO fully asynchronous (i.e. does not implement the Akka Persistence [snapshot store plugin API](http://doc.akka.io/docs/akka/2.3.6/scala/persistence.html#snapshot-store-plugin-api) directly).

### Configuration

To activate the snapshot-store plugin, add the following line to your Akka `application.conf`:

    akka.persistence.snapshot-store.plugin = "cassandra-snapshot-store"

This will run the snapshot store with its default settings. The default settings can be changed with the following configuration keys:

- `cassandra-snapshot-store.contact-points`. A comma-separated list of contact points in a Cassandra cluster. Default value is `[127.0.0.1]`. Host:Port pairs are also supported. In that case the port parameter will be ignored.
- `cassandra-snapshot-store.port`. Port to use to connect to the Cassandra host. Default value is `9042`. Will be ignored if the contact point list is defined by host:port pairs.
- `cassandra-snapshot-store.keyspace`. Name of the keyspace to be used by the plugin. If the keyspace doesn't exist it is automatically created. Default value is `akka_snapshot`.
- `cassandra-snapshot-store.table`. Name of the table to be used by the plugin. If the table doesn't exist it is automatically created. Default value is `snapshots`.
- `cassandra-snapshot-store.replication-factor`. Replication factor to use when a keyspace is created by the plugin. Default value is `1`.
- `cassandra-snapshot-store.max-metadata-result-size`. Maximum number of snapshot metadata to load per recursion (when trying to find a snapshot that matches specified selection criteria). Default value is `10`. Only increase this value when selection criteria frequently select snapshots that are much older than the most recent snapshot i.e. if there are much more than 10 snapshots between the most recent one and selected one. This setting is only for increasing load efficiency of snapshots.
- `cassandra-snapshot-store.write-consistency`. Write consistency level. Default value is `ONE`.
- `cassandra-snapshot-store.read-consistency`. Read consistency level. Default value is `ONE`.

To connect to the Cassandra hosts with credentials, add the following lines:

- `cassandra-snapshot-store.authentication.username`. The username to use to login to Cassandra hosts. No authentication is set as default.
- `cassandra-snapshot-store.authentication.password`. The password corresponding to username. No authentication is set as default.

To limit the Cassandra hosts this plugin connects with to a specific datacenter, use the following setting:

- `cassandra-snapshot-store.local-datacenter`.  The id for the local datacenter of the Cassandra hosts that this module should connect to.  By default, this property is not set resulting in Datastax's standard round robin policy being used.
