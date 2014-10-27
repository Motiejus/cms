.. _cloud:

Running CMS in the cloud
========================

By design, CMS is a scalable distributed system. However, it is not ready to run
in a dynamic cloud environment out of the box: a few tweaks are necessary:

1. Initial setup (provisiniong phase).
2. Configuration file management.

Initial setup
-------------

Like seen in :ref:`installation_dependencies`, installation is non-trivial.
However, the installation process is easy to abstract in a `Docker
<http://docker.com>` container.

A CMS Docker container in planned. A single container instance will support all
available service types (``Worker``, ``ContestWebServer``, ``LogService``,
etc). Since codebase is identical in all container types, the only difference
lies in which service ought to be started. Therefore, type of the spawned
container can be handled a layer above.

These different machine configurations would therefore exist on the cloud:

* PostgreSQL machine(s).
* ``N`` machines for ``Worker``.
* ``M`` machines for ``ContestWebServer``.
* 1 machine for all the rest of the (core) services.

Database provisioning is not CMS-specific, therefore it makes sense to use a
hosted database solution (e.g. `AWS RDS`). Generic database maintenance, like
auto-scaling, backups, failover will be handled by the database service
provider, rather than contest operator. Given that, databases are out of the
scope of this document.

Network configuration
---------------------

Network setup is not straightforward. Docker assigns IP addresses for its
instances from the same pool, and it is likely that two containers on different
hosts will have the same IP of ``eth0``.

That violates a few assumptions of ``ResourceService``. ``ResourceService``
decides which services to start by matching host's IP address(es) with the
addresses in the configuration.

Configuration files
-------------------

CMS is configured by two files, identical on all machines: :file:`cms.conf` and
:file:`cms.ranking.conf`. In case there are changes in topology, currently
there are no standardized means to update the files on all nodes and restart the
affected services.

The solution suggests itself: have means to update the files and restart the
services.

Implementation sketch
---------------------

Goal of this exercise: be able to provision CMS on an EC2 cluster without
connecting to any server. Tools:

* *A priori* provisioned empty postgresql DB (credentials known).
* `Docker`_. A self-contained CMS image which can be deployed on different
  environments.
* `Docker registry`_. Place to build and store the Docker image.
* `etcd`_. Distributed configuration daemon. Stores state and configuration of
  our cluster.
* `cloud-init`_. Helps to initially configure Cloud virtual machines.
* `CoreOS`_. Low-footprint Linux OS with Docker and etcd
  installed.

High steps of a cluster provisioning:

1. In AWS management console, user selects "CoreOS" instance type with user
   data filled in.  User data will contain:

   * Types of containers to spawn, can be >1 (``Worker``, ``LogService``, etc).
   * Initial values for :file:`cms.conf` and :file:`cms.ranking.conf` (except
     hosts). *Optional*.

   Note: can be more than one host.
2. CoreOS will create or join an etcd cluster. Read more in `CoreOS AWS docs`_.
3. Container(s) is (are) started like defined in userdata.
4. Container starts supervisord, which starts cms-etcd (a script yet to be
   written).
5. cms-etcd will connect to etcd and "agree" on ``cms.conf`` and
   ``cms.ranking.conf``. That "agreed" file written to the database. Details on
   how "agreement" is achieved will follow. cms-etcd keeps running and monitoring
   etcd for any changes.
6. cms-etcd instructs supervisord to (re)start ResourceService.
7. ResourceService starts CMS with the up-to-date configuration.

When anything changes in etcd (user-supplied configuration or a cluster
topology change), steps 5-7 are repeated.

Configuration storage in etcd
-----------------------------

Example etcd state::

  /
  ├── configs/
  │   ├── 172.17.0.1_worker_0001/
  │   │   ├── cms/
  │   │   │   ├── database
  │   │   │   ├── temp_dir
  │   │   │   └── ...
  │   │   └── ranking/
  │   │       └── ...
  │   ├── 172.17.0.1_worker_0005/
  │   │   ├── cms/
  │   │   |   └── ...
  │   │   └── ranking/
  │   │       └── ...
  │   ├── 172.17.0.2_logservice_0001/
  │   └── ...
  └── current -> configs/172.17.0.2_logservice_0001/

Directories under ``configs/*/cms/`` and ``configs/*/ranking`` have a layout
very similar to :file:`cms.conf`, and :file:`cms.ranking.conf`, respectively.
Mapping from the etcd tree to :file:`cms.conf` and :file:`cms.ranking.conf` is
straightforward.

``/current`` is the reference to the subdirectory currently employed by the
cluster.

Configuration change process:

1. Node takes a snapshot of ``current`` directory, saving its name.
2. Alters the configuration (locally) as needed.
3. Pushes the whole folder to etcd. This is always possible, because directory
   names are globally unique (has a node identifier and a counter).
4. Compare-And-Set ``current`` to the name of the most recently pushed directory.
   Previous value for CAS is the name saved in (1).
5.
   1. If CAS succeeds, cms-etcd stores the new configuration file and instructs
         supervisord to restart ResourceService.

   2. If CAS does not succeed, the process is restarted from step 1.

I do not know the formal language to prove that the FSM above has no livelocks
or deadlocks. With a stable cluster, however, liveness is guaranteed, because
upon each re-started transaction a node in the cluster "promotes" itself.

Docker
------

This part abstracts system configuration and installation of CMS.

To keep network setup simple, we will allow containers to use host's network
stack: ``--net=host``.

To be continued...

.. _`AWS RDS`: http://aws.amazon.com/rds/
.. _`Docker`: https://docker.com/
.. _`Docker registry`: https://registry.hub.docker.com/
.. _`etcd`: https://coreos.com/using-coreos/etcd/
.. _`cloud-init`: http://cloudinit.readthedocs.org/
.. _`CoreOS`: https://coreos.com/
.. _`CoreOS AWS docs`: https://coreos.com/docs/running-coreos/cloud-providers/ec2/#cloud-config
