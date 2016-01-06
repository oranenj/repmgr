==========================================
repmgr: Replication Manager for PostgreSQL
==========================================

`repmgr` is a suite of open-source tools to manage replication and failover
within a cluster of PostgreSQL servers. It enhances PostgreSQL's built-in
replication capabilities with utilities to set up standby servers, monitor
replication, and perform administrative tasks such as failover or manual
switchover operations.


Overview
========

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

For a multi-master replication solution, please see 2ndQuadrant's BDR (bi-directional
replication) extension. For selective replication, e.g. of individual tables
or databases from one server to another, please see 2ndQuadrant's pglogical extension.


Concepts
--------

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



Metadata
--------

In order to effectively manage a replication cluster, `repmgr` needs to store
information about the servers in the cluster in a dedicated database schema.
This schema is automatically created during the first step in initialising
a `repmgr`-controlled cluster (`repmgr primary register`) and contains the
following objects:

tables:
  - repl_events:
  - repl_nodes:
  - repl_monitor:

views:
  - repl_status:
  - repl_show_nodes:


The `repmgr` metadata schema can be stored in an existing database or in its own
dedicated database.


Installation
============

System requirements
-------------------

`repmgr` is developed and tested on Linux and OS X, but should work on any
UNIX-like system supported by PostgreSQL.

`repmgr` supports PostgreSQL from version 9.3.

All servers in the replication cluster must be running the same major version of
PostgreSQL, and we recommend that they also run the same minor version.

`repmgr` must be installed on each server in the replication cluster.

A dedicated system user for `repmgr` is *not* required; as many `repmgr` and
`repmgrd` actions require direct access to the PostgreSQL data directory,
it is usually executed by the `postgres` user.

Packages
--------

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


Source installation
-------------------

`repmgr` source code can be obtained directly from the project GitHub repository:

    git clone https://github.com/2ndQuadrant/repmgr

Release tarballs are also available:

    https://github.com/2ndQuadrant/repmgr/releases
    http://repmgr.org/downloads.php

`repmgr` is compiled in the same way as a PostgreSQL extension using the PGXS
infrastructure, e.g.:

    sudo make USE_PGXS=1 install



Setting up a replication cluster with repmgr
============================================




Configuration
-------------

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
