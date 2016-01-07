repmgr: Replication Manager for PostgreSQL
==========================================

`repmgr` is a suite of open-source tools to manage replication and failover
within a cluster of PostgreSQL servers. It enhances PostgreSQL's built-in
replication capabilities with utilities to set up standby servers, monitor
replication, and perform administrative tasks such as failover or manual
switchover operations.


Overview
--------

The `repmgr` suite provides two main tools:

- `repmgr` - a command-line tool used to perform administrative tasks such as:
    - setting up standby servers
    - promoting a standby server to primary
    - switching over primary and standby servers
    - displaying the status of servers in the replication cluster

- `repmgrd` is a daemon which actively monitors servers in a replication cluster:
    - monitoring and recording replication performance
    - performing failover by detecting failure of the primary and
      promoting the most suitable standby server
    - provide notifications about events in the cluster to a user-defined
      script which can perform tasks such as sending alerts by email


`repmgr` supports and enhances PostgreSQL's built-in streaming replication, which
provides a single read/write primary server and one or more read-only standbys
containing near-real time copies of the primary server's database.

For a multi-master replication solution, please see 2ndQuadrant's BDR
(bi-directional replication) extension. For selective replication, e.g.
of individual tables or databases from one server to another, please
see 2ndQuadrant's pglogical extension.


### Concepts

- replication cluster

In the `repmgr` documentation, "replication cluster" refers to the network
of PostgreSQL servers connected by streaming replication.

- primary server

The primary server (sometimes called "master server") is the only server in the
replication cluster in read/write mode. Changes made to the primary server
are propagated to connected standby servers.

- standby server

All other servers in the replication cluster are standby servers (sometimes
called "slaves" or "replicas") in read-only mode. By default, replication
to standby servers is asynchronous, meaning standby servers will generally
be slightly behind changes to the primary server. It's also possible to
define one standby server as a synchronous standby which is always
in the same state as the primary server, though this can mean commits
on the primary server are delayed until confirmed on the synchronous standby.

- cascading replication

Standby servers can connect to another standby server instead of the primary,
enabling changes on the primary to be cascaded down through a hierarchy of
standby servers.

- failover

If a primary server fails, generally it's desirable to promote a suitable
standby as the new primary. The `repmgrd` daemon supports automatic failover
in this kind of case.

- switchover

In certain circumstances, such as hardware or operating system maintenance,
it's necessary to take a primary server offline; in this case a controlled
switchover is necessary, whereby a suitable standby is promoted and the
existing primary removed from the replication cluster in a controlled manner.
The `repmgr` command line client provides this functionality.

- witness server

`repmgr` provides functionality to set up a so-called "witness server" to
assist in determining a new primary server in a failover situation with more
than one standby. The witness server itself is not part of the replication
cluster, although it does contain a copy of the repmgr metadata schema
(see below).

The purpose of a witness server is to provide a "casting vote" where servers
in the replication cluster are split over more than one location. In the event
of a loss of connectivity between locations, the presence or absence of
the witness server will decide whether a server at that location is promoted
to primary; this is to prevent a "split-brain" situation where an isolated
location interprets a network outage as a failure of the (remote) primary and
promotes a (local) standby.

A witness server only needs to be created if `repmgrd` is in use.

### repmgr user and metadata

In order to effectively manage a replication cluster, `repmgr` needs to store
information about the servers in the cluster in a dedicated database schema.
This schema is automatically created during the first step in initialising
a `repmgr`-controlled cluster (`repmgr primary register`) and contains the
following objects:

tables:
  - repl_events: events of interest
  - repl_nodes: connection and status information for each server in the
    replication cluster
  - repl_monitor: historical standby monitoring information written by `repmgrd`

views:
  - repl_show_nodes: based on the `repl_nodes` showing name of the server's
    upstream node
  - repl_status: when `repmgrd`'s monitoring is enabled, shows current monitoring
    status for each node

The `repmgr` metadata schema can be stored in an existing database or in its own
dedicated database.

A dedicated superuser is required to own the meta-database as well as carry out
administrative actions.

Installation
------------

### System requirements

`repmgr` is developed and tested on Linux and OS X, but should work on any
UNIX-like system supported by PostgreSQL itself.

`repmgr` supports PostgreSQL from version 9.3.

All servers in the replication cluster must be running the same major version of
PostgreSQL, and we recommend that they also run the same minor version.

`repmgr` must be installed on each server in the replication cluster.

A dedicated system user for `repmgr` is *not* required; as many `repmgr` and
`repmgrd` actions require direct access to the PostgreSQL data directory,
it is usually executed by the `postgres` user.

Additionally, we recommend installing `rsync` and enabling passwordless
`ssh` connectivity between all servers in the replication cluster.

### Packages

We recommend installing `repmgr` using the available packages for your
system.

- RedHat/CentOS: RPM packages for `repmgr` are available via Yum throug
  the PostgreSQL Global Development Group RPM repository ( http://yum.postgresql.org/ ).
  You need to follow the instructions for your distribution (RedHat, CentOS,
  Fedora, etc.) and architecture as detailed at yum.postgresql.org.

- Debian/Ubuntu: the most recent `repmgr` packages are available from the
  PostgreSQL Community APT repository ( http://apt.postgresql.org/ ).
  Instructions can be found in the APT section of the PostgreSQL Wiki
  ( https://wiki.postgresql.org/wiki/Apt ).

See `PACKAGES.md` for details on building .deb and .rpm packages
from the `repmgr` source code.


### Source installation

`repmgr` source code can be obtained directly from the project GitHub repository:

    git clone https://github.com/2ndQuadrant/repmgr

Release tarballs are also available:

    https://github.com/2ndQuadrant/repmgr/releases
    http://repmgr.org/downloads.php

`repmgr` is compiled in the same way as a PostgreSQL extension using the PGXS
infrastructure, e.g.:

    sudo make USE_PGXS=1 install

`repmgr` can be built in any environment suitable for building PostgreSQL itself.


### Configuration

`repmgr` and `repmgrd` use a common configuration file, usually called `repmgr.conf`
(although any name can be used if explicitly specified). At the very least,
`repmgr.conf` must contain connection parameters for the local database.

The configuration file will be looked for in the following locations:

- a configuration file specified by the `-f/--config-file` command line option
- `repmgr.conf` in the local directory
- `/etc/repmgr.conf`
- the directory reported by `pg_config --sysconfdir`

Note that if a file is explicitly specified with `-f/--config-file`, an error will
be raised if it is not found or not readable and no attempt will be made to check
default locations.

For a full list of annotated configuration items, see the file `repmgr.conf.sample`.

Certain items in the configuration file can be overridden with command line options:

- `-L/--log-level`
- `-b/--pg_bindir`


Setting up a simple replication cluster with repmgr
---------------------------------------------------

The following section will describe how to set up a basic replication cluster
with a primary and a standby server using the `repmgr` command line tool.
It is assumed PostgreSQL is installed on both servers in the cluster,
`rsync` is available and password-less SSH connections are possible between
both servers.

    TIP: for testing `repmgr`, it's possible to use multiple PostgreSQL
    instances running on different ports on the same computer, with
    password-less SSH access to `localhost` enabled.

### PostgreSQL configuration

On the primary server, a PostgreSQL instance must be initialised and operational.
The following replication settings must be included in `postgresql.conf`:

    # Allow read-only queries on standby servers. The number of WAL
    # senders should be larger than the number of standby servers.

    hot_standby = on
    wal_level = 'hot_standby'
    max_wal_senders = 10

    # How much WAL to retain on the primary to allow a temporarily
    # disconnected standby to catch up again. The larger this is, the
    # longer the standby can be disconnected. This is needed only in
    # 9.3; from 9.4, replication slots can be used instead (see below).

    wal_keep_segments = 5000

    # Enable archiving, but leave it unconfigured (so that it can be
    # configured without a restart later). Recommended, not required.

    archive_mode = on
    archive_command = 'cd .'

Create a dedicated PostgreSQL superuser account and a database for
the `rempgr` metadata, e.g.

    createuser -s repmgr
    createdb repmgr -O repmgr

For the examples in this document, the name `repmgr` will be used for both
user and database, but any names can be used.

Ensure the `repmgr` user has appropriate permissions in `pg_hba.conf` and
can connect in replication mode; `pg_hba.conf` should contain entries
similar to the following:

    local   replication   repmgr                              trust
    host    replication   repmgr      127.0.0.1/32            trust
    host    replication   repmgr      192.168.1.0/32          trust

    local   repmgr        repmgr                              trust
    host    repmgr        repmgr      127.0.0.1/32            trust
    host    repmgr        repmgr      192.168.1.0/32          trust

Adjust according to your network environment and authentication requirements.

On the standby, do not create a PostgreSQL instance, but do ensure an empty
directory is available for the `postgres` system user to create a data
directory.


### repmgr configuration file

Create a `repmgr.conf` file on both servers. The file must contain at
least the following parameters:

    cluster=test
    node=1
    node_name=node1
    conninfo='host=repmgr_node1 user=repmgr dbname=repmgr'

- `cluster`: an arbitrary name for the replication cluster; this must be identical
     on all nodes
- `node`: a unique integer identifying the node
- `node_name`: a unique string identifying the node; we recommend a name
     specific to the server (e.g. 'server_1'); avoid names indicating the
     current replication role like 'primary' or 'standby' as the server's
     role could change.
- `conninfo`: a valid connection string for the `repmgr` database on the
     *current* server. (On the standby, the database will not yet exist, but
     `repmgr` needs to know the connection details to complete the setup
     process).

`repmgr.conf` should not be stored inside the PostgreSQL data directory,
as it could be overwritten when setting up or reinitialising the PostgreSQL
server. See section `Configuration` above for further details about `repmgr.conf`.

`repmgr` will create a schema named after the cluster and prefixed with `repmgr_`,
e.g. `repmgr_test`; we also recommend that you set the `repmgr` user's search path
to include this schema name, e.g.

    ALTER USER repmgr SET search_path TO repmgr_test, "$user", public;

### Initialise the primary server

To enable `repmgr` to support a replication cluster, the primary node must
be registered with `repmgr`, which creates the `repmgr` database:

    $ repmgr -f repmgr.conf primary register
    [2016-01-07 16:56:46] [NOTICE] master node correctly registered for cluster test with id 1 (conninfo: host=repmgr_node1 user=repmgr dbname=repmgr)

### Clone the standby server

The next step, cloning the standby from the primary server, is where `repmgr`
starts to unfold its true potential, by combining a number of otherwise manual
steps into one command, `repmgr standby clone`.

    $ repmgr -h repmgr_node1 -U repmgr -d repmgr -D /path/to/node2/data/ standby clone
    [2016-01-07 17:21:26] [NOTICE] destination directory '/path/to/node2/data/' provided
    [2016-01-07 17:21:26] [NOTICE] starting backup...
    [2016-01-07 17:21:26] [HINT] this may take some time; consider using the -c/--fast-checkpoint option
    NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
    [2016-01-07 17:21:28] [NOTICE] standby clone (using pg_basebackup) complete
    [2016-01-07 17:21:28] [NOTICE] you can now start your PostgreSQL server
    [2016-01-07 17:21:28] [HINT] for example : pg_ctl -D /path/to/node2/data/ start

This will clone the PostgreSQL data directory files from the primary, including
`postgresql.conf` and `pg_hba.conf` files, and additionally automatically create
the `recovery.conf` file containing the correct parameters to start streaming
from the primary server. Make any adjustments to the configuration files now, then
start the standby PostgreSQL server.

### Verify replication is functioning

Connect to the primary server and execute:

    repmgr=# SELECT * FROM pg_stat_replication ;
    -[ RECORD 1 ]----+------------------------------
    pid              | 7704
    usesysid         | 16384
    usename          | repmgr
    application_name | walreceiver
    client_addr      | 192.168.1.2
    client_hostname  |
    client_port      | 46196
    backend_start    | 2016-01-07 17:32:58.322373+09
    backend_xmin     |
    state            | streaming
    sent_location    | 0/3000220
    write_location   | 0/3000220
    flush_location   | 0/3000220
    replay_location  | 0/3000220
    sync_priority    | 0
    sync_state       | async


Reference
---------

### Error codes

`repmgr` or `repmgrd` will return one of the following error codes on program
exit:

* SUCCESS (0)              Program ran successfully.
* ERR_BAD_CONFIG (1)       Configuration file could not be parsed or was invalid
* ERR_BAD_RSYNC (2)        An rsync call made by the program returned an error
* ERR_NO_RESTART (4)       An attempt to restart a PostgreSQL instance failed
* ERR_DB_CON (6)           Error when trying to connect to a database
* ERR_DB_QUERY (7)         Error while executing a database query
* ERR_PROMOTED (8)         Exiting program because the node has been promoted to master
* ERR_BAD_PASSWORD (9)     Password used to connect to a database was rejected
* ERR_STR_OVERFLOW (10)    String overflow error
* ERR_FAILOVER_FAIL (11)   Error encountered during failover (repmgrd only)
* ERR_BAD_SSH (12)         Error when connecting to remote host via SSH
* ERR_SYS_FAILURE (13)     Error when forking (repmgrd only)
* ERR_BAD_BASEBACKUP (14)  Error when executing pg_basebackup
* ERR_MONITORING_FAIL (16) Unrecoverable error encountered during monitoring (repmgrd only)

Support and Assistance
----------------------

2ndQuadrant provides 24x7 production support for repmgr, including
configuration assistance, installation verification and training for
running a robust replication cluster. For further details see:

* http://2ndquadrant.com/en/support/

There is a mailing list/forum to discuss contributions or issues
http://groups.google.com/group/repmgr

The IRC channel #repmgr is registered with freenode.

Further information is available at http://www.repmgr.org/

We'd love to hear from you about how you use repmgr. Case studies and
news are always welcome. Send us an email at info@2ndQuadrant.com, or
send a postcard to

    repmgr
    c/o 2ndQuadrant
    7200 The Quorum
    Oxford Business Park North
    Oxford
    OX4 2JZ
    United Kingdom

Thanks from the repmgr core team.

* Ian Barwick
* Jaime Casanova
* Abhijit Menon-Sen
* Simon Riggs
* Cedric Villemain

Further reading
---------------

* http://blog.2ndquadrant.com/announcing-repmgr-2-0/
* http://blog.2ndquadrant.com/managing-useful-clusters-repmgr/
* http://blog.2ndquadrant.com/easier_postgresql_90_clusters/
