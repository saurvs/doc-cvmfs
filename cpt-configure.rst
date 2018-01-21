Client Configuration
====================

Structure of /etc/cvmfs
-----------------------

The local configuration of CernVM-FS is controlled by several files in
``/etc/cvmfs`` listed in the table below. For every .conf file
except for the files in /etc/cvmfs/default.d you can create a
corresponding .local file having the same prefix in order to customize
the configuration. The .local file will be sourced after the
corresponding .conf file.

In a typical installation, a handful of parameters need to be set in
/etc/cvmfs/default.local. Most likely, this is the list of repositories
(``CVMFS_REPOSITORIES``), HTTP proxies (see :ref:sct_network`),
and perhaps the cache directory and the cache quota (see
:ref:sct_cache`) In a few cases, one might change a parameter
for a specific domain or a specific repository, provide an exclusive cache for
a specific repository (see :ref:sct_cache`). For a list of all
parameters, see Appendix ":ref:`apxsct_clientparameters`".

The .conf and .local configuration files are key-value pairs in the form
``PARAMETER=value``. They are sourced by /bin/sh. Hence, a limited set
of shell commands can be used inside these files including comments,
``if`` clauses, parameter evaluation, and shell math (``$((...))``).
Special characters have to be quoted. For instance, instead of
``CVMFS_HTTP_PROXY=p1;p2``, write ``CVMFS_HTTP_PROXY='p1;p2'`` in order
to avoid parsing errors. The shell commands in the configuration files
can use the ``CVMFS_FQRN`` parameter, which contains the fully qualified
repository names that is being mounted. The current working directory is
set to the parent directory of the configuration file at hand.

.. _tab_configfiles:

============================== =================================================
**File**                       **Purpose**
------------------------------ -------------------------------------------------
``config.sh``                  Set of internal helper functions.
``default.conf``               Set of base parameters.
``default.d/$config.conf``     Adjustments to the default.conf configuration,
                               usually installed by a cvmfs-config-...
                               package. Read before default.local.
``domain.d/$domain.conf``      Domain-specific parameters and implementations
                               of the functions in ``config.sh``
``config.d/$repository.conf``  Repository-specific parameters and
                               implementations of the functions in ``config.sh``
``keys/``                      Contains domain-specific sub directories with
                               public keys used to verify the digital signature
                               of file catalogs
============================== =================================================

.. _sct_config_repository:

The "Config Repository"
~~~~~~~~~~~~~~~~~~~~~~~

In addition to the local system configuration, a client can configure a
dedicated "config repository". A config repository is a standard
mountable CernVM-FS repository that resembles the directory structure of
/etc/cvmfs. It can be used to centrally maintain the public keys and
configuration of repositories that should not be distributed with rather
static packages, and also to centrally
:ref:`blacklist <sct_blacklisting>` compromised repository keys.
Configuration from the config repository is overwritten
by the local configuration in case of conflicts; see the comments in
/etc/cvmfs/default.conf for the precise ordering of processing
the config files.  The config repository
is set by the ``CVMFS_CONFIG_REPOSITORY`` parameter. The default
configuration rpm cvmfs-config-default sets this parameter to
cvmfs-config.cern.ch.

The ``CVMFS_CONFIG_REPO_REQUIRED`` parameter can be used to force availability
of the config repository in order for other repositories to get mounted.

Mounting
--------

Mounting of CernVM-FS repositories is typically handled by autofs. Just
by accessing a repository directory under /cvmfs (/cvmfs/atlas.cern.ch),
autofs will take care of mounting. autofs will also automatically
unmount a repository if it is not used for a while.

Instead of using autofs, CernVM-FS repositories can be mounted manually
with the system's ``mount`` command. In order to do so, use the
``cvmfs`` file system type, like

::

      mount -t cvmfs atlas.cern.ch /cvmfs/atlas.cern.ch

Likewise, CernVM-FS repositories can be mounted through entries in
/etc/fstab. A sample entry in /etc/fstab:

::

      atlas.cern.ch /mnt/test cvmfs defaults,_netdev,nodev 0 0

Every mount point corresponds to a CernVM-FS process. Using autofs or
the system's mount command, every repository can only be mounted once.
Otherwise multiple CernVM-FS processes would collide in the same cache
location. If a repository is needed under several paths, use a *bind
mount* or use a :ref:`private file system mount point <sct_privatemount>`.

.. _sct_privatemount:

Private Mount Points
~~~~~~~~~~~~~~~~~~~~

In contrast to the system's ``mount`` command which requires root
privileges, CernVM-FS can also be mounted like other Fuse file systems
by normal users. In this case, CernVM-FS uses parameters from one or
several user-provided config files instead of using the files under
/etc/cvmfs. CernVM-FS private mount points do not appear as ``cvmfs2``
file systems but as ``fuse`` file systems. The ``cvmfs_config`` and
``cvmfs_talk`` commands ignore privately mounted CernVM-FS repositories.
On an interactive machine, private mount points are for instance
unaffected by an administrator unmounting all system's CernVM-FS mount
points by ``cvmfs_config umount``.

In order to mount CernVM-FS privately, use the ``cvmfs2`` command like

::

      cvmfs2 -o config=myparams.conf atlas.cern.ch /home/user/myatlas

A minimal sample myparams.conf file could look like this:

::

      CVMFS_CACHE_BASE=/home/user/mycache
      CVMFS_RELOAD_SOCKETS=/home/user/mycache
      CVMFS_USYSLOG=/home/user/cvmfs.log
      CVMFS_CLAIM_OWNERSHIP=yes
      CVMFS_SERVER_URL=http://cvmfs-stratum-one.cern.ch/cvmfs/atlas.cern.ch
      CVMFS_KEYS_DIR=/etc/cvmfs/keys/cern.ch
      CVMFS_HTTP_PROXY=DIRECT

Make sure to use absolute path names for the mount point and for the
cache directory. Use ``fusermount -u`` in order to unmount a privately
mounted CernVM-FS repository.

The private mount points can also be used to use the CernVM-FS Fuse
module in case it has not been installed under /usr and /etc. If the
public keys are not installed under /etc/cvmfs/keys, the directory of
the keys needs to be specified in the config file by
``CVMFS_KEYS_DIR=<directory>``. If the libcvmfs\_fuse.so library is not
installed in one of the standard search paths, the ``LD_LIBRARY_PATH``
variable has to be set accordingly for the ``cvmfs2`` command.

Docker Containers
~~~~~~~~~~~~~~~~~

There are two options to mount CernVM-FS in docker containers. The first
option is to bind mount a mounted repository as a volume into the
container. This has the advantage that the CernVM-FS cache is shared
among multiple containers. The second option is to mount a repository
inside a container, which requires a *privileged* container.

Volume Driver
^^^^^^^^^^^^^
There is an `external package <https://gitlab.cern.ch/cloud-infrastructure/docker-volume-cvmfs/>`_
that provides a Docker Volume Driver for CernVM-FS.
This package provides management of repositories in Docker and Kubernetes.
It provides a convenient interface to handle CernVM-FS volume definitions.

Bind mount from the host
^^^^^^^^^^^^^^^^^^^^^^^^

On Docker >= 1.10, the autofs managed area /cvmfs can be directly mounted into
the container as a shared mount point like

::

    docker run -it -v /cvmfs:/cvmfs:shared centos /bin/bash

In order to bind mount an individual repository from the host, turn off autofs
on the host and mount the repository manually, like:

::

    service autofs stop  # systemd: systemctl stop autofs
    chkconfig autofs off  # systemd: systemctl disable autofs
    mkdir -p /cvmfs/sft.cern.ch
    mount -t cvmfs sft.cern.ch /cvmfs/sft.cern.ch

Start the docker container with the ``-v`` option to mount the
CernVM-FS repository inside, like

::

    docker run -it -v /cvmfs/sft.cern.ch:/cvmfs/sft.cern.ch centos /bin/bash

The ``-v`` option can be used multiple times with different
repositories.

Mount inside a container
^^^^^^^^^^^^^^^^^^^^^^^^

In order to use ``mount`` inside a container, the container must be
started in privileged mode, like

::

        docker run --privileged -i -t centos /bin/bash

In such a container, CernVM-FS can be installed and used the usual way
provided that autofs is turned off.

Parrot Connector to CernVM-FS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In case Fuse cannot be be installed, the `parrot toolkit
<http://ccl.cse.nd.edu/software/parrot>`_ provides a means to "mount"
CernVM-FS on Linux in pure user space.
Parrot sandboxes an application in a similar way gdb sandboxes an
application. But instead of debugging the application,
parrot transparently rewrites file system calls and can effectively
provide /cvmfs to an application. We recommend to use the `latest
precompiled parrot <http://ccl.cse.nd.edu/software/downloadfiles.php>`_, which
has CernVM-FS support built-in.

In order to sandbox a command ``<CMD>`` with options ``<OPTIONS>`` in
parrot, use

::

    export PARROT_ALLOW_SWITCHING_CVMFS_REPOSITORIES=yes
    export PARROT_CVMFS_REPO="<default-repositories>"
    export HTTP_PROXY='<SITE HTTP PROXY>'  # or 'DIRECT;' if not on a cluster or grid site
    parrot_run <PARROT_OPTIONS> <CMD> <OPTIONS>

Repositories that are not available by default from the builtin
``<default-repositories>`` list can be explicitly added to
``PARROT_CVMFS_REPO``. The repository name, a stratum 1 URL, and the
public key of the repository need to be provided. For instance, in order
to add alice-ocdb.cern.ch and ilc.desy.de to the list of repositories,
one can write

::

    export CERN_S1="http://cvmfs-stratum-one.cern.ch/cvmfs"
    export DESY_S1="http://grid-cvmfs-one.desy.de:8000/cvmfs"
    export PARROT_CVMFS_REPO="<default-repositories> \
      alice-ocdb.cern.ch:url=${CERN_S1}/alice-ocdb.cern.ch,pubkey=<PATH/key.pub> \
      ilc.desy.de:url=${DESY_S1}/ilc.desy.de,pubkey=<PATH/key.pub>"

given that the repository public keys are in the provided paths.

By default, parrot uses a shared CernVM-FS cache for all parrot
instances of the same user stored under a temporary directory that is
derived from the user id. In order to place the CernVM-FS cache into a
different directory, use

::

    export PARROT_CVMFS_ALIEN_CACHE=</path/to/cache>

In order to share this directory among multiple users, the users have to
belong to the same UNIX group.

.. _sct_network:

Network Settings
----------------

CernVM-FS uses HTTP for the data transfer. Repository data can be
replicated to multiple web servers and cached by standard web proxies
such as Squid [Guerrero99]_. In a typical setup, repositories are replicated to
a handful of web servers in different locations. These replicas form the
CernVM-FS Stratum 1 service, whereas the replication source server is
the CernVM-FS Stratum 0 server. In every cluster of client machines,
there should be two or more web proxy servers that CernVM-FS can use
(see :ref:`cpt_squid`). These site-local web proxies reduce the
network latency for the CernVM-FS clients and they reduce the load for
the Stratum 1 service. CernVM-FS supports WPAD/PAC proxy auto
configuration [Gauthier99]_, choosing a random proxy for load-balancing, and
automatic fail-over to other hosts and proxies in case of network
errors. Roaming clients can connect directly to the Stratum 1 service.

IP Protocol Version
~~~~~~~~~~~~~~~~~~~

CernVM-FS can use both IPv4 and IPv6. For dual-stack stratum 1 hosts it will use
the system default settings when connecting directly to the host. When
connecting to a proxy, by default it will try on the IPv4 address unless the
proxy only has IPv6 addresses configured. The ``CVMFS_IPFAMILY_PREFER=[4|6]``
parameter can be used to select the preferred IP protocol for dual-stack
proxies.

Stratum 1 List
~~~~~~~~~~~~~~

To specify the Stratum 1 servers, set ``CVMFS_SERVER_URL`` to a
semicolon-separated list of known replica servers (enclose in quotes).
The so defined URLs are organized as a ring buffer. Whenever download of
files fails from a server, CernVM-FS automatically switches to the next
mirror server. For repositories under the cern.ch domain, the Stratum 1
servers are specified in /etc/cvmfs/domain.d/cern.ch.conf.

It is recommended to adjust the order of Stratum 1 servers so that the closest
servers are used with priority. This can be done automatically by :ref:`using
geographic ordering <sct_geoapi>`. Alternatively, for roaming
clients (clients not using a proxy server), the Stratum 1 servers can be
automatically sorted according to round trip time by ``cvmfs_talk host probe``
(see :ref:`sct_tools`). Otherwise, the proxy server would invalidate round
trip time measurement.

The special sequence ``@fqrn@`` in the ``CVMFS_SERVER_URL`` string is
replaced by fully qualified repository name (atlas.cern.cn, cms.cern.ch,
...). That allows to use the same parameter for many repositories hosted
under the same domain. For instance,
http://cvmfs-stratum-one.cern.ch/cvmfs/@fqrn@ can resolve to
http://cvmfs-stratum-one.cern.ch/cvmfs/atlas.cern.ch,
http://cvmfs-stratum-one.cern.ch/cvmfs/cms.cern.ch, and so on depending
on the repository that is being mounted. The same works for the sequence
``@org@`` which is replaced by the unqualified repository name (atlas,
cms, ...).

Proxy Lists
~~~~~~~~~~~

CernVM-FS uses a dedicated HTTP proxy configuration, independent from
system-wide settings. Instead of a single proxy, CernVM-FS uses a *chain
of load-balanced proxy groups*. The CernVM-FS proxies are set by the
``CVMFS_HTTP_PROXY`` parameter.

Proxies within the same proxy group are considered as a load-balance
group and a proxy is selected randomly. If a proxy fails,
CernVM-FS automatically switches to another proxy from the current
group. If all proxies from a group have failed, CernVM-FS switches to
the next proxy group. After probing the last proxy group in the chain,
the first proxy is probed again. To avoid endless loops, for each file
download the number of switches is restricted by the total number of
proxies.

The chain of proxy groups is specified by a string of semicolon
separated entries, each group is a list of pipe separated
hostnames [#]_. Multiple IP addresses behind a single proxy host name
(DNS *round-robin* entry) are automatically transformed into a
load-balanced group.  In order to limit the number of proxy servers used from
a round-robin DNS entry, set ``CVMFS_MAX_IPADDR_PER_PROXY``.  This also limits
the perceived "hang duration" while CernVM-FS performs fail-overs.

The ``DIRECT`` keyword for a hostname avoids using proxies. Note that the
``CVMFS_HTTP_PROXY`` parameter is necessary in order to mount. If you don't use
proxies, set the parameter to ``DIRECT``.

Multiple proxy groups are often organized as a primary proxy group at
the local site and backup proxy groups at remote sites. In order to
avoid CernVM-FS being stuck with proxies at a remote site after a
fail-over, CernVM-FS will automatically retry to use proxies from the
primary group after some time. The delay for re-trying a proxies from
the primary group is set in seconds by ``CVMFS_PROXY_RESET_AFTER``. The
distinction of primary and backup proxy groups can be turned off by
setting this parameter to 0.


Automatic Proxy Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The proxy settings can be automatically gathered through WPAD. The
special proxy server "auto" in ``CVMFS_HTTP_PROXY`` is resolved
according to the proxy server specification loaded from a PAC file. PAC
files can be on a file system or accessible via HTTP. CernVM-FS looks
for PAC files in the order given by the semicolon separated URLs in the
``CVMFS_PAC_URLS`` environment variable. This variable defaults to
http://wpad/wpad.dat. The ``auto`` keyword used as a URL in
``CVMFS_PAC_URLS`` is resolved to http://wpad/wpad.dat, too, in order to
be compatible with Frontier [Blumenfeld08]_.

Fallback Proxy List
~~~~~~~~~~~~~~~~~~~

In addition to the regular proxy list set by ``CVMFS_HTTP_PROXY``, a
fallback proxy list is supported in ``CVMFS_FALLBACK_PROXY``. The syntax
of both lists is the same. The fallback proxy list is appended to the
regular proxy list, and if the fallback proxy list is set, any DIRECT is
removed from both lists. The automatic proxy configuration of the
previous section only sets the regular proxy list, not the fallback
proxy list. Also the fallback proxy list can be automatically reordered;
see the next section.

.. _sct_geoapi:

Ordering of Servers according to Geographic Proximity
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CernVM-FS Stratum 1 servers provide a RESTful service for geographic
ordering. Clients can request
`http://<HOST>/cvmfs/<FQRN>/api/v1.0/geo/<proxy\_address>/<server\_list>`
The proxy address can be replaced by a UUID if no proxies are used, and
the CernVM-FS client does that if there are no regular proxies. The
server list is comma-separated. The result is an ordered list of indexes
of the input host names. Use of this API can be enabled in a
CernVM-FS client with ``CVMFS_USE_GEOAPI=yes``. That will geographically
sort both the servers set by ``CVMFS_SERVER_URL`` and the fallback
proxies set by ``CVMFS_FALLBACK_PROXY``.

Timeouts
~~~~~~~~

CernVM-FS tries to gracefully recover from broken network links and
temporarily overloaded paths. The timeout for connection attempts and
for very slow downloads can be set by ``CVMFS_TIMEOUT`` and
``CVMFS_TIMEOUT_DIRECT``. The two timeout parameters apply to a
connection with a proxy server and to a direct connection to a Stratum 1
server, respectively. A download is considered to be "very slow" if the
transfer rate is below for more than the timeout interval. The threshold
can be adjusted with the ``CVMFS_LOW_SPEED_LIMIT`` parameter. A very
slow download is treated like a broken connection.

On timeout errors and on connection failures (but not on name resolving
failures), CernVM-FS will retry the path using an exponential backoff.
This introduces a jitter in case there are many concurrent requests by a
cluster of nodes, allowing a proxy server or web server to serve all the
nodes consecutively. ``CVMFS_MAX_RETRIES`` sets the number of retries on
a given path before CernVM-FS tries to switch to another proxy or host.
The overall number of requests with a given proxy/host combination is
``$CVMFS_MAX_RETRIES``\ +1. ``CVMFS_BACKOFF_INIT`` sets the maximum
initial backoff in seconds. The actual initial backoff is picked with
milliseconds precision randomly in the interval
:math:`[1, \text{\$CVMFS\_BACKOFF\_INIT}\cdot 1000]`. With every retry,
the backoff is then doubled.

.. _sct_cache:

Cache Settings
--------------

Downloaded files will be stored in a local cache directory. The
CernVM-FS cache has a soft quota; as a safety margin, the partition
hosting the cache should provide more space than the soft quota limit.
Once the quota limit is reached, CernVM-FS will automatically remove
files from the cache according to the least recently used policy.
Removal of files is performed bunch-wise until half of the maximum cache
size has been freed. The quota limit can be set in Megabytes by
``CVMFS_QUOTA_LIMIT``. For typical repositories, a few Gigabytes make a
good quota limit.

The cache directory needs to be on a local file system in order to allow
each host the accurate accounting of the cache contents; on a network
file system, the cache can potentially be modified by other hosts.
Furthermore, the cache directory is used to create (transient) sockets
and pipes, which is usually only supported by a local file system. The
location of the cache directory can be set by ``CVMFS_CACHE_BASE``.

On SELinux enabled systems, the cache directory and its content need to
be labeled as ``cvmfs_cache_t``. During the installation of
CernVM-FS RPMs, this label is set for the default cache directory
/var/lib/cvmfs. For other directories, the label needs to be set
manually by ``chcon -Rv --type=cvmfs_cache_t $CVMFS_CACHE_BASE``.

Each repository can either have an exclusive cache or join the
CernVM-FS shared cache. The shared cache enforces a common quota for all
repositories used on the host. File duplicates across repositories are
stored only once in the shared cache. The quota limit of the shared
directory should be at least the maximum of the recommended limits of
its participating repositories. In order to have a repository not join
the shared cache but use an exclusive cache, set
``CVMFS_SHARED_CACHE=no``.

Alien Cache
~~~~~~~~~~~

An "alien cache" provides the possibility to use a data cache outside
the control of CernVM-FS. This can be necessary, for instance, in HPC
environments where local disk space is not available or scarce but
powerful cluster file systems are available. The alien cache directory
is a directory in addition to the ordinary cache directory. The ordinary
cache directory is still used to store control files.

The alien cache directory is set by the ``CVMFS_ALIEN_CACHE`` option. It
can be located anywhere including cluster and network file systems. If
configured, all data chunks are stored there. CernVM-FS ensures atomic
access to the cache directory. It is safe to have the alien directory
shared by multiple CernVM-FS processes and it is safe to unlink files
from the alien cache directory anytime. The contents of files, however,
must not be touched by third-party programs.

In contrast to normal cache mode where files are store in mode 0600, in
the alien cache files are stored in mode 0660. So all users being part
of the alien cache directory's owner group can use it.

The skeleton of the alien cache directory should be created upfront.
Otherwise, the first CernVM-FS process accessing the alien cache
determines the ownership. The ``cvmfs2`` binary can create such a
skeleton using

::

    cvmfs2 __MK_ALIEN_CACHE__ $alien_cachedir $owner_uid $owner_gid

Since the alien cache is unmanaged, there is no automatic quota
management provided by CernVM-FS; the alien cache directory is
ever-growing. The ``CVMFS_ALIEN_CACHE`` requires
``CVMFS_QUOTA_LIMIT=-1`` and ``CVMFS_SHARED_CACHE=no``.

The alien cache might be used in combination with a special repository
replication mode that preloads a cache directory
(Section :ref:`cpt_replica`). This allows to propagate an entire repository
into the cache of a cluster file system for HPC setups that do not allow
outgoing connectivity.

.. _sct_cache_advanced:

Advanced Cache Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For exotic cache configurations, CernVM-FS supports specifying multiple,
independent "cache manager instances" of different types. Such cache manager
instances replace the local cache directory. Since the local cache directory is
also used to store transient special files, ``CVMFS_WORKSPACE=$local_path``
must be used when advanced cache configuration is used.

A concrete cache manager instance has a user-defined name and it is specified
like

::

    CVMFS_CACHE_PRIMARY=myInstanceName
    CVMFS_CACHE_myInstanceName_TYPE=posix

Multiple instances can thus be safely defined with different names but only one
is selected when the client boots. The following table lists the valid cache
manager instance types.

=========== ======================================================================
** Type**   **Behavior**
=========== ======================================================================
posix       Uses a cache directory with the standard cache implementation
tiered      Uses two other cache manager instances in a layered configuration
external    Uses an external cache plugin process (see Section :ref:`cpt_plugins`)
=========== ======================================================================

The instance name "default" is blocked because the regular cache configuration
syntax is automatically mapped to ``CVMFS_CACHE_default_...`` parameters.  The
command ``sudo cvmfs_talk cache instance`` can be used to show the currently
used cache manager instance.


Tiered Cache
^^^^^^^^^^^^

The tiered cache manager combines two other cache manager instances as an upper
layer and a lower layer into a single functional cache manager.  Usually, a
small and fast upper layer (SSD, memory) is combined with a larger and slower
lower layer (HDD, network drive). The upper layer needs to be large enough to
serve all currently open files.  On an upper layer cache miss, CernVM-FS tries
to copy the missing object from the lower into the upper layer. On a lower layer
cache miss, CernVM-FS download and stores objects either in both layers or in
the upper layer only, depending on the configuration.

The parameters ``CVMFS_CACHE_$tieredInstanceName_UPPER`` and
``CVMFS_CACHE_$tieredInstanceName_LOWER`` set the names of the upper and the
lower instances.  The parameter
``CVMFS_CACHE_$tieredInstanceName_LOWER_READONLY=[yes|no]`` controls whether the
lower layer can be populated by the client or not.



External Cache Plugin
^^^^^^^^^^^^^^^^^^^^^

A CernVM-FS cache manager instance can be provided by an external process. The
cache manager process and the CernVM-FS client are connected through a socket,
whose address is called "locator". The locator can either address a UNIX domain
socket on the local file system, or a TCP socket, as in the following examples

::

    CVMFS_CACHE_instanceName_LOCATOR=unix=/var/lib/cvmfs/cache.socket
    # or
    CVMFS_CACHE_instanceName_LOCATOR=tcp=192.168.0.24:4242

If a UNIX domain socket is used, both the CernVM-FS client and the cache manager
need to be able to access the socket file. Usually that means they have to run
under the same user.

Instead of manually starting the cache manager, the CernVM-FS client can
optionally automatically start and stop the cache manager process. This is
called a "supervised cache manager". The first booting CernVM-FS client starts
the cache manager process, the last terminating client stops the cache manager
process. In order to start the cache manager in supervised mode, use
``CVMFS_CACHE_instanceName_CMDLINE=<executable and arguments>``, using a comma
(``,``) instead of a space to separate the command line parameters.


.. _sct_cache_advanced_example:

Example
^^^^^^^

The following example configures a tiered cache with an external cache plugin
as an upper layer and a read-only, network drive as a lower layer. The cache
plugin uses memory to cache data and is part of the CernVM-FS client. This
configuration could be used in a data center with diskless nodes and a preloaded
cache on a network drive (see Chapter :ref:`cpt_hpc`)

::

    CVMFS_WORKSPACE=/var/lib/cvmfs
    CVMFS_CACHE_PRIMARY=hpc

    CVMFS_CACHE_hpc_TYPE=tiered
    CVMFS_CACHE_hpc_UPPER=memory
    CVMFS_CACHE_hpc_LOWER=preloaded
    CVMFS_CACHE_hpc_LOWER_READONLY=yes

    CVMFS_CACHE_memory_TYPE=external
    CVMFS_CACHE_memory_CMDLINE=/usr/libexec/cvmfs/cache/cvmfs_cache_ram,/etc/cvmfs/cache-mem.conf
    CVMFS_CACHE_memory_LOCATOR=unix=/var/lib/cvmfs/cvmfs-cache.socket

    CVMFS_CACHE_preloaded_TYPE=posix
    CVMFS_CACHE_preloaded_ALIEN=/gpfs/cvmfs/alien
    CVMFS_CACHE_preloaded_SHARED=no
    CVMFS_CACHE_preloaded_QUOTA_LIMIT=-1

The example configuration for the in-memory cache plugin in
/etc/cvmfs/cache-mem.conf is

::

    CVMFS_CACHE_PLUGIN_LOCATOR=unix=/var/lib/cvmfs/cvmfs-cache.socket
    # 2G RAM
    CVMFS_CACHE_PLUGIN_SIZE=2000


NFS Server Mode
---------------

In case there is no local hard disk space available on a cluster of
worker nodes, a single CernVM-FS client can be exported via
nfs [Callaghan95]_ [Shepler03]_ to these worker nodes.This mode of deployment
will inevitably introduce a performance bottleneck and a single point of
failure and should be only used if necessary.

NSF export requires Linux kernel >= 2.6.27 on the NFS server. For
instance, exporting works for Scientific Linux 6 but not for Scientific
Linux 5. The NFS server should run a lock server as well. For proper NFS
support, set ``CVMFS_NFS_SOURCE=yes``. On the client side, all available nfs
implementations should work.

In the NFS mode, upon mount an additionally directory
nfs\_maps.$repository\_name appears in the CernVM-FS cache directory.
These *NFS maps* use leveldb to store the virtual inode CernVM-FS issues
for any accessed path. The virtual inode may be requested by NFS clients
anytime later. As the NFS server has no control over the lifetime of
client caches, entries in the NFS maps cannot be removed.

Typically, every entry in the NFS maps requires some 150-200 Bytes. A
recursive ``find`` on /cvmfs/atlas.cern.ch with 50 million entries, for
instance, would add up 8GB in the cache directory. For a CernVM-FS instance
that is exported via NFS, the safety margin for the NFS maps needs be
taken into account. It also might be necessary to monitor the actual
space consumption.

Tuning
~~~~~~

The default settings in CernVM-FS are tailored to the normal, non-NFS
use case. For decent performance in the NFS deployment, the amount of
memory given to the meta-data cache should be increased. By default,
this is 16M. It can be increased, for instance, to 256M by setting
``CVMFS_MEMCACHE_SIZE`` to 256. Furthermore, the maximum number of
download retries should be increased to at least 2.

The number of NFS daemons should be increased as well. A value of 128
NFS daemons has shown perform well. In Scientific Linux, the number of
NFS daemons is set by the ``RPCNFSDCOUNT`` parameter in
/etc/sysconfig/nfs.

The performance will benefit from large RAM on the NFS server
(:math:`\geq` 16GB) and CernVM-FS caches hosted on an SSD
hard drive.

Shared NFS Maps (HA-NFS)
~~~~~~~~~~~~~~~~~~~~~~~~

As an alternative to the existing, `leveldb
<https://github.com/google/leveldb>`_ managed NFS maps, the NFS
maps can optionally be managed out of the CernVM-FS cache directory by
SQLite. This allows the NFS maps to be placed on shared storage and
accessed by multiple CernVM-FS NFS export nodes simultaneously for
clustering and active high-availablity setups. In order to enable shared
NFS maps, set ``CVMFS_NFS_SHARED`` to the path that should be used to
host the SQLite database. If the path is on shared storage, the shared
storage has to support POSIX file locks. The drawback of the
SQLite managed NFS maps is a significant performance penalty which in
practice can be covered by the memory caches.

Example
~~~~~~~

An example entry /etc/exports (note: the fsid needs to be different for
every exported CernVM-FS repository)

::

      /cvmfs/atlas.cern.ch 172.16.192.0/24(ro,sync,no_root_squash,\
        no_subtree_check,fsid=101)

A sample entry /etc/fstab entry on a client:

::

      172.16.192.210:/cvmfs/atlas.cern.ch /cvmfs/atlas.cern.ch nfs4 \
        ro,ac,actimeo=60,lookupcache=all,nolock,rsize=1048576,wsize=1048576 0 0

.. _sct_hotpatch:

Hotpatching and Reloading
-------------------------

By hotpatching a running CernVM-FS instance, most of the code can be
reloaded without unmounting the file system. The current active code is
unloaded and the code from the currently installed binaries is loaded.
Hotpatching is logged to syslog. Since CernVM-FS is re-initialized
during hotpatching and configuration parameters are re-read, hotpatching
can be also seen as a "reload".

Hotpatching has to be done for all repositories concurrently by

::

      cvmfs_config [-c] reload

The optional parameter ``-c`` specifies if the CernVM-FS cache should be
wiped out during the hotpatch. Reloading of the parameters of a specific
repository can be done like

::

      cvmfs_config reload atlas.cern.ch

In order to see the history of loaded CernVM-FS Fuse modules, run

::

      cvmfs_talk hotpatch history

The currently loaded set of parameters can be shown by

::

      cvmfs_talk parameters

The CernVM-FS packages use hotpatching in the package upgrade process.

.. _sct_tools:

Auxiliary Tools
---------------

cvmfs\_fsck
~~~~~~~~~~~

CernVM-FS assumes that the local cache directory is trustworthy.
However, it might happen that files get corrupted in the cache directory
caused by errors outside the scope of CernVM-FS. CernVM-FS stores files
in the local disk cache with their cryptographic content hash key as
name, which makes it easy to verify file integrity. CernVM-FS contains
the ``cvmfs_fsck`` utility to do so for a specific cache directory. Its
return value is comparable to the system's ``fsck``. For example,

::

      cvmfs_fsck -j 8 /var/lib/cvmfs/shared

checks all the data files and catalogs in ``/var/lib/cvmfs/shared``
using 8 concurrent threads. Supported options are:

================ ===============================================================
``-v``           Produce more verbose output.
``-j #threads``  Sets the number of concurrent threads that check files in the
                 cache directory. Defaults to 4.
``-p``           Tries to automatically fix problems.
``-f``           Unlinks the cache database. The database will be automatically
                 rebuilt by CernVM-FS on next mount.
================ ===============================================================

The ``cvmfs_config fsck`` command can be used to verify all configured
repositories.

cvmfs\_config
~~~~~~~~~~~~~

The ``cvmfs_config`` utility provides commands in order to setup the
system for use with CernVM-FS.

**setup**
    The ``setup`` command takes care of basic setup tasks, such as
    creating the cvmfs user and allowing access to CernVM-FS mount
    points by all users.

**chksetup**
    The ``chksetup`` command inspects the system and the
    CernVM-FS configuration in /etc/cvmfs for common problems.

**showconfig**
    The ``showconfig`` command prints the CernVM-FS parameters for all
    repositories or for the specific repository given as argument.  With the
    `-s` option, only non-empty parameters are shown.

**stat**
    The ``stat`` command prints file system and network statistics for
    currently mounted repositories.

**status**
    The ``status`` command shows all currently mounted repositories and
    the process id (PID) of the CernVM-FS processes managing a mount
    point.

**probe**
    The ``probe`` command tries to access /cvmfs/$repository for all
    repositories specified in ``CVMFS_REPOSITORIES`` or the ones specified as
    a space separated list on the command line, respectively.

**fsck**
    Run ``cvmfs_fsck`` on all repositories specified in ``CVMFS_REPOSITORIES``.

**reload**
    The ``reload`` command is used to :ref:`reload or hotpatch
    CernVM-FS instances <sct_hotpatch>`.

**umount**
    The ``umount`` command unmounts all currently mounted
    CernVM-FS repositories, which will only succeed if there are no open
    file handles on the repositories.

**wipecache**
    The ``wipecache`` command is an alias for ``reload -c``.

**killall**
    The ``killall`` command immediately unmounts all repositories under
    /cvmfs and terminates the associated processes.  It is meant to escape from
    a hung state without the need to reboot a machine.  However, all processes
    that use CernVM-FS at the time will be terminated, too.  The need to use
    this command very likely points to a network problem or a bug in cvmfs.

**bugreport**
    The ``bugreport`` command creates a tarball with collected system
    information which helps to :ref:`debug a problem <sct_debugginghints>`.

cvmfs\_talk
~~~~~~~~~~~

The ``cvmfs_talk`` command provides a way to control a currently running
CernVM-FS process and to extract information about the status of the
corresponding mount point. Most of the commands are for special purposes
only or covered by more convenient commands, such as
``cvmfs_config showconfig`` or ``cvmfs_config stat``. Three commands might
be of particular interest though.

::

      cvmfs_talk cleanup 0

will, without interruption of service, immediately cleanup the cache
from all files that are not currently pinned in the cache.

::

      cvmfs_talk cleanup rate 120

shows the number of cache cleanups in the last two hours (120 minutes).  If
this value is larger than one or two, the cache size is probably two small and
the client experiences cache thrashing.

::

      cvmfs_talk internal affairs

prints the internal status information and performance counters. It can
be helpful for performance engineering.

Other
~~~~~

Information about the current cache usage can be gathered using the
``df`` utility. For repositories created with the CernVM-FS 2.1
toolchain, information about the overall number of file system entries
in the repository as well as the number of entries covered by currently
loaded meta-data can be gathered by ``df -i``.

For the `Nagios monitoring system <http://www.nagios.org>`_ [Schubert08]_, a
checker plugin is available `on our website
<http://cernvm.cern.ch/portal/filesystem/downloads>`_.

Debug Logs
----------

The ``cvmfs2`` binary forks a watchdog process on start. Using this
watchdog, CernVM-FS is able to create a stack trace in case certain
signals (such as a segmentation fault) are received. The watchdog writes
the stack trace into syslog as well as into a file ``stacktrace`` in the
cache directory.

In addition to :ref:`these debugging hints <sct_debugginghints>`, CernVM-FS
can be started in debug mode. In the debug mode, CernVM-FS will log with high
verbosity which makes the debug mode unsuitable for production use. In order
to turn on the debug mode, set ``CVMFS_DEBUGFILE=/tmp/cvmfs.log``.


.. rubric:: Footnotes

.. [#]
   The usual proxy notation rules apply, like
   ``http://proxy1:8080|http://proxy2:8080;DIRECT``
