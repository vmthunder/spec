..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cache group support
===============================================================================

https://blueprints.launchpad.net/nova/+spec/add-cachegroup-support

Local cache is desirable when we do read/write with remote storage servers. For
example, we can use local disks as cache for remote volumes. We propose to add
cache support in nova to make use of local storage of the compute nodes.

Problem description
===================

Currently, there is no general cache support for block device in nova, and data
is stored in the cinder server volumes which are attached to the compute nodes.
We propose to cache data on local storage devices (HDDs and SSDs) of the compute
nodes, so that they can access data in their local cache instead of in the
remote cinder servers. To achieve this goal, there are some challenges.

1.  Since compute nodes dynamically attach and release volumes from servers,
    the cache scheme must support dynamically changing configurations, which
    supports to add and remove disks freely.
2.  The cache scheme for block device should support different kinds of cache
    modules (e.g. bcache, dm-cache, flashcache, lvm-cache).

Use Cases
----------
Cache should be transparent so that users can use the cache the same as the cinder
volume.

Project Priority
-----------------
undefined

Proposed change
===============

Since this cache is on the compute-side, we add it into a cache group after a volume
is attached, and create a volume with cache for future use. To realize this,
some modifications in nova are needed.

1.  We need to add a parameter to attach_volume (...) which indicates whether use
    cache or not. If yes, add the volume to cache group after it's attached.
2.  The cache modules (bcache, dm-cache, etc.) should be added to the drivers.
3.  Nova should setup a database record to indicate which volume is cached.
4.  When detach a volume, we should remove the cache first.
5.  To support dynamically cache scheme, and add/remove disk freely, we need to
    organize the cached volume as a group. For bcache, backing devices can be
    attached and detached at runtime. We can also add bcache under same cache
    group interface for uniformity.

Since we have already implemented the grouping functionality for flashcache,
called flashcachegroup, abbreviated as fcg, as an example to illustrate the
procedure. Please refer to https://github.com/lihuiba/flashcachegroup for
detail.

*  Fcg uses dm-linear to create a logical group of HDDs and combine all SSDs.
*  Fcg makes cache of logical HDD group with the linear combined SSDs,
   called cached group.
*  When adding HDD to the logical HDD group, fcg splits a cached HDD out of
   the cached group by using dm-linear accordingly.
*  When removing HDD from the logical HDD group, fcg also removes the cached
   HDD accordingly.

Besides flashcache, we can also implement dm-cache and others following the same
procedure. And lvm is also good idea instead of dmsetup. We can put this
fcg-like stuff into cinder drivers if needed.


Alternatives
------------

DM-Cache
DM-Cache uses I/O scheduling and cache management techniques optimized for
flash-based SSDs. The device mapper target (DM-Cache) reuses the metadata
library used in the thin-provisioning library. Both write-back and
write-through are supported by DM-Cache. The problem of DM-Cache is that its
metadata device is not easy to handle, and SSD still cannot be removed freely
so far.

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

The performance of accessing to the volume with local cache will improve.

Other deployer impact
---------------------
To use cachegroup, config "nova.conf" set "use_cachegroup = true".

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee: vmThunderGroup (vmthunder)

Work Items
----------

Add config option and relevant DB in nova
Add cachegroup implement code in nova
Add unit and integrated tests


Dependencies
============

Dependencies depends on the specific cache schemes.
For using flashcachegroup, Facebook’s flashcache must already be installed.
For bcachegroup, Linux kernel must >= 3.10

Testing
=======

The unit tests and integrated tests will be added to the component.

Documentation Impact
====================
Using the cachegroup will be documented.


References
==========

Flashcachegroup: https://github.com/lihuiba/flashcachegroup
