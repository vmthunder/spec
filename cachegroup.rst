..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cache group support
===============================================================================

https://blueprints.launchpad.net/nova/+spec/add-cachegroup-support

Local cache is desirable when we do read/write with remote storage servers. We
propose to add cache support in nova to use the local storage of compute nodes
as cache for remote cinder volumes.

Problem description
===================

Currently, there is no general cache support for block devices in nova, and data
is stored in the volumes of cinder servers which are attached to the compute nodes.
This may result in severe bottleneck of I/O performance. To address this problem,
we propose CacheGroup, which caches data on local storage devices of the compute
nodes, so that they can access data in their local cache instead of in the remote
cinder servers. Several challenges have to be addressed to achieve this goal.

1.  Since compute nodes dynamically attach and release volumes from cinder
    servers, the cache scheme must support dynamically changing configurations,
    which supports to add and remove disks freely.
2.  The cache scheme for block devices should support different kinds of cache
    modules (e.g., bcache, dm-cache, flashcache, lvm-cache).

Use Cases
----------
Cache should be transparent so that users can use the cache in the same way that
they use the cinder volume.

Project Priority
-----------------
undefined

Proposed change
===============

We implement CacheGroup as a package, and to use its caching functionality some 
modifications in nova are needed.
1.  We need to add a parameter to attach_volume(...) which indicates whether to
    use cache or not. If the parameter is true, add the volume to cache group after
    it is attached.
2.  The cache modules (flashcache, bcache, etc.) should be added to the drivers.
3.  Nova should setup a database record to indicate which volume is cached.
4.  When detach a volume, we should remove the cache first.
5.  To support dynamically cache scheme, and add/remove disk freely, we need to
    organize the cached volume as a group. Backing devices can be attached and
    detached at runtime.

CacheGroup is implemented by grouping both the remote storages and the local
caches. Since we have already implemented the functionality for flashcache, 
we take FlashCacheGroup (fcg) as an example to illustrate the details.

*  Fcg uses dm-linear to create a logical group for remote volumes and combine
   the local storages (HDDs or SSDs).
*  Fcg makes cache of the logical volume group using the combined local storage,
   called cached group.
*  When adding a new remote volume to the logical volume group, fcg create a 
   corresponding cached volume out of the cached group using dm-linear.
*  When removing a volume from the logical volume group, fcg also removes the cached
   volume accordingly.

CacheGroup can be implemented for other caches (e.g., bcache and dm-cache) following
the same procedure.

Alternatives
------------

DM-Cache
DM-Cache uses I/O scheduling and cache management techniques optimized for
flash-based SSDs. The device mapper target (DM-Cache) reuses the metadata
library used in the thin-provisioning library. Both write-back and
write-through are supported by DM-Cache. The problem of DM-Cache is that its
metadata device is not easy to handle.

LVM-Cache
LVM-Cache is built on top of DM-Cache so that logical volumes can be turned into
cache devices. Because of that, LVM-Cache splits the cache pool LV into two
devices - the cache data LV and cache metadata LV. LVM-Cache will face the same
problem with DM-Cache.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

The performance of accessing remote cinder volumes will improve.

Other deployer impact
---------------------
To use cachegroup, config "nova.conf" set "use_cachegroup = true".

Developer impact
----------------

None

Implementation
==============

We have already implemented the cache group functionality for flashcache.
See https://github.com/lihuiba/flashcachegroup for details.

Assignee(s)
-----------

Primary assignee: Ziyang Li

Work Items
----------

Add config option and relevant DB in nova
Add cachegroup implement code in nova
Add unit and integrated tests


Dependencies
============

Dependencies depends on the specific cache schemes.
For using flashcachegroup, Facebook’s flashcache must already be installed.
For bcachegroup, Linux kernel >= 3.10

Testing
=======

The unit tests and integrated tests will be added to the component.

Documentation Impact
====================
Using the cachegroup will be documented.


References
==========

Flashcachegroup: https://github.com/lihuiba/flashcachegroup
