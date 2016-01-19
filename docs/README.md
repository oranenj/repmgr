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

- `repmgrd` is a daemon which actively monitors servers in a replication cluster
   and performs the following tasks:
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

This guide assumes that you are familiar with PostgreSQL administration and
streaming replication concepts. For further details on streaming
replication, see this link:

  http://www.postgresql.org/docs/current/interactive/warm-standby.html#STREAMING-REPLICATION

The following terms are used throughout the `repmgr` documentation.

- `replication cluster`

In the `repmgr` documentation, "replication cluster" refers to the network
of PostgreSQL servers connected by streaming replication.

- `node`

A `node` is a server within a network cluster.

- `upstream node`

This is the node a standby server is connected to; either the primary server or in
the case of cascading replication, another standby.

- `failover`

If a primary server fails, generally it's desirable to promote a suitable
standby as the new primary. The `repmgrd` daemon supports automatic failover
in this kind of case.

- `switchover`

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
  - `repl_events`: records events of interest
  - `repl_nodes`: connection and status information for each server in the
    replication cluster
  - `repl_monitor`: historical standby monitoring information written by `repmgrd`

views:
  - `repl_show_nodes`: based on the `repl_nodes` showing name of the server's
    upstream node
  - `repl_status`: when `repmgrd`'s monitoring is enabled, shows current monitoring
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

The `repmgr` tools must be installed on each server in the replication cluster.

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

`repmgr` and `repmgrd` use a common configuration file, usually named
`repmgr.conf` (although any name can be used if explicitly specified).
At the very least, `repmgr.conf` must contain the connection parameters
for the local `repmgr` database.

The configuration file will be looked for in the following locations:

- a configuration file specified by the `-f/--config-file` command line option
- `repmgr.conf` in the local directory
- `/etc/repmgr.conf`
- the directory reported by `pg_config --sysconfdir`

Note that if a file is explicitly specified with `-f/--config-file`, an error will
be raised if it is not found or not readable and no attempt will be made to check
default locations.

For a full list of annotated configuration items, see the file `repmgr.conf.sample`.

These parameters in the configuration file can be overridden with command line
options:

- `-L/--log-level`
- `-b/--pg_bindir`


Setting up a simple replication cluster with repmgr
---------------------------------------------------

The following section will describe how to set up a basic replication cluster
with a primary and a standby server using the `repmgr` command line tool.
It is assumed PostgreSQL is installed on both servers in the cluster,
`rsync` is available and password-less SSH connections are possible between
both servers.

    *TIP*: for testing `repmgr`, it's possible to use multiple PostgreSQL
    instances running on different ports on the same computer, with
    password-less SSH access to `localhost` enabled.

### PostgreSQL configuration

On the primary server, a PostgreSQL instance must be initialised and running.
The following replication settings must be included in `postgresql.conf`:

    # Ensure WAL files contain enough information to enable read-only queries
    # on the standby

    wal_level = 'hot_standby'

    # Enable up to 10 replication connections

    max_wal_senders = 10

    # How much WAL to retain on the primary to allow a temporarily
    # disconnected standby to catch up again. The larger this is, the
    # longer the standby can be disconnected. This is needed only in
    # 9.3; from 9.4, replication slots can be used instead (see below).

    wal_keep_segments = 5000

    # Enable read-only queries on a standby
    # (Note: this will be ignored on a primary but we recommend including
    # it anyway)

    hot_standby = on


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

Create a `repmgr.conf` file on the primary server. The file must contain at
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
be registered with `repmgr`, which creates the `repmgr` database and adds
a metadata record for the server:

    $ repmgr -f repmgr.conf primary register
    [2016-01-07 16:56:46] [NOTICE] master node correctly registered for cluster test with id 1 (conninfo: host=repmgr_node1 user=repmgr dbname=repmgr)

The metadata record looks like this:

    repmgr=# SELECT * FROM repmgr_test.repl_nodes;
     id |  type   | upstream_node_id | cluster | name  |                  conninfo                   | slot_name | priority | active
    ----+---------+------------------+---------+-------+---------------------------------------------+-----------+----------+--------
      1 | master  |                  | test    | node1 | host=repmgr_node1 dbname=repmgr user=repmgr |           |      100 | t
    (1 row)

Each server in the replication cluster will have its own record and will be updated
when its status or role changes.

### Clone the standby server

Create a `repmgr.conf` file on the standby server. It must contain at
least the same parameters as the primary's `repmgr.conf`, but with
the values `node`, `node_name` and `conninfo` adjusted accordingly, e.g.:

    cluster=test
    node=2
    node_name=node2
    conninfo='host=repmgr_node2 user=repmgr dbname=repmgr'

Clone the standby with:

    $ repmgr -h repmgr_node1 -U repmgr -d repmgr -D /path/to/node2/data/ -f /path/to/node2/repmgr.conf standby clone
    [2016-01-07 17:21:26] [NOTICE] destination directory '/path/to/node2/data/' provided
    [2016-01-07 17:21:26] [NOTICE] starting backup...
    [2016-01-07 17:21:26] [HINT] this may take some time; consider using the -c/--fast-checkpoint option
    NOTICE:  pg_stop_backup complete, all required WAL segments have been archived
    [2016-01-07 17:21:28] [NOTICE] standby clone (using pg_basebackup) complete
    [2016-01-07 17:21:28] [NOTICE] you can now start your PostgreSQL server
    [2016-01-07 17:21:28] [HINT] for example : pg_ctl -D /path/to/node2/data/ start

This will clone the PostgreSQL data directory files from the primary using
PostgreSQL's pg_basebackup utility. A `recovery.conf` file containing the
correct parameters to start streaming from the primary server will
be created automatically, and unless otherwise the `postgresql.conf` and `pg_hba.conf`
files will be copied.

Make any adjustments to the configuration files now, then start the standby server.

*NOTE*: `repmgr standby clone` does not require `repmgr.conf`, however we
recommend providing this as `repmgr` will set the `application_name` parameter
in `recovery.conf` as value provided in `node_name`, making it easier to identify
the node in `pg_stat_replication`. It's also possible to provide some advanced
options for controlling the standby cloning process; see next section for
details.

### Verify replication is functioning

Connect to the primary server and execute:

    repmgr=# SELECT * FROM pg_stat_replication;
    -[ RECORD 1 ]----+------------------------------
    pid              | 7704
    usesysid         | 16384
    usename          | repmgr
    application_name | node2
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


### Register the standby

Register the standby server with:

    repmgr -f /path/to/node_2/repmgr.conf standby register
    [2016-01-19 11:13:16] [NOTICE] standby node correctly registered for cluster test with id 2 (conninfo: host=repmgr_node2 user=repmgr dbname=repmgr)

Connect to the standby servers' `repmgr` database and check the `repl_nodes`
table:

    repmgr=# SELECT * FROM repmgr_test.repl_nodes;

     id |  type   | upstream_node_id | cluster | name  |                  conninfo                   | slot_name | priority | active
    ----+---------+------------------+---------+-------+---------------------------------------------+-----------+----------+--------
      1 | master  |                  | test    | node1 | host=repmgr_node1 dbname=repmgr user=repmgr |           |      100 | t
      2 | standby |                1 | test    | node2 | host=repmgr_node2 dbname=repmgr user=repmgr |           |      100 | t
    (2 rows)

The standby server now has a copy of records for all servers in the replication
cluster.


Advanced options for cloning a standby
--------------------------------------

The above section demonstrates the simplest possible way to clone
a standby server. Depending on your situation, finer-grained control
over the cloning process may be necessary.

### pg_basebackup options when cloning a standby

By default, `pg_basebackup` performs a checkpoint before beginning the
backup process. However, a normal checkpoint may take some time to complete;
a fast checkpoint can be forced with `repmgr`'s  `-c/--fast-checkpoint`
option. This may impact performance of the server being cloned from
so should be used with care.

Further options can be passed to the `pg_basebackup` utility via
the `pg_basebackup_options` in `repmgr.conf`. See the PostgreSQL
documentation for more details:
  http://www.postgresql.org/docs/current/static/app-pgbasebackup.html

### Using rsync to clone a standby

By default `repmgr` uses the `pg_basebackup` utility to clone a standby's
data directory from the primary. Under some circumstances it may be
desirable to use `rsync` to do this, such as when resyncing the data
directory of a failed server with an active replication node.

To use `rsync` instead of `pg_basebackup`, provide the `-r/--rsync-only`
option when executing `repmgr standby clone`.

Note that `repmgr` forces `rsync` to use `--checksum` mode to ensure that all
the required files are copied. This results in additional I/O on both source
and destination server as the contents of files existing on both servers need
to be compared, meaning this method is not necessarily faster than making a
fresh clone with `pg_basebackup`.


### Dealing with configuration files

By default, `repmgr` will attempt to copy the standard configuration files
(`postgresql.conf`, `pg_hba.conf` and `pg_ident.conf`) even if they are located
outside of the data directory (though note currently they will be copied
into the standby's data directory). To prevent this happening, when executing
`repmgr standby clone` provide the `--ignore-external-config-files` option.

If using `rsync` to clone a standby, additional control over which files
not to transfer is possible by configuring `rsync_options` in `repmgr.conf`,
which enables any valid `rsync` options to be passed to that command, e.g.:

    rsync_options='--exclude=postgresql.local.conf'


Using replication slots with repmgr
-----------------------------------

http://www.postgresql.org/docs/current/interactive/warm-standby.html#STREAMING-REPLICATION-SLOTS


Setting up cascading replication with repmgr
--------------------------------------------




Promoting a standby server with repmgr
--------------------------------------

If a primary server fails or needs to be removed from the replication cluster,
a standby needs to be promoted to become the new primary server.


Performing a switchover with repmgr
-----------------------------------


Removing a standby from a replication cluster
---------------------------------------------


Automatic failover with repmgrd
-------------------------------


Using a witness server with repmgrd
------------------------------------


Generating event notifications with repmgr/repmgrd
--------------------------------------------------


Upgrading repmgr
----------------

`repmgr` is updated regularly with point releases (e.g. 3.0.2 to 3.0.3)
containing bugfixes and other minor improvements. Any substantial new
functionality will be included in a feature release (e.g. 3.0.x to 3.1.x).

In general `repmgr` can be upgraded as-is without any further action required,
however feature releases may require the `repmgr` database to be upgraded.
An SQL script will be provided - please check the release notes for details.

Reference
---------

### repmgr commands

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

* http://blog.2ndquadrant.com/managing-useful-clusters-repmgr/
* http://blog.2ndquadrant.com/easier_postgresql_90_clusters/
