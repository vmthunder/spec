..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cache support
===============================================================================

https://blueprints.launchpad.net/cinder/+spec/add-cachegroup-support

Conventional hard disk drive (HDD) has become a performance bottleneck compared
with CPU, memory and network. High-end storage (solid state drive, even memory)
is much faster than HDD, but has lower capacity, shorter lifetime and higher
price issues. Some block-level caching solutions (flashcache, bcache, dm-cache,
lvm-cache) enhance storage performance by using a fast device as cache of the
slow devices. We propose to add cache support in cinder to make use of local
storage in compute node.

Problem description
===================

Currently, there is no general cache support for block device in Cinder. Data is
stored in volumes of cinder servers and attached to compute nodes. To take full
advantage of local storage devices (HDDs and SSDs) of the compute nodes, we can
cache data on them. Thus, compute nodes can access data in local cache before
send I/O requests to servers. To achieve this goal, there are some challenges.

1.  Since compute nodes dynamically attach and release volumes from servers,
    the cache scheme must support dynamically changing configurations, which
    supports to add and remove disks freely.
2.  The cache scheme for block device should support different kinds of cache
    modules (e.g. flashcache and bcache).

Use Cases
----------
As a deployer, make remote attached volume be cached in computer nodes can
improve the performance of compute nodes effectively and ease the pressure of
the storage server.

Project Priority
-----------------
undefined

Proposed change
===============

Since this cache is on compute-side, we add it into cache group after a volume
is attached, and create a volume with cache for future use. To realize this, some
modifications in cinder are needed.

We propose the following changes:

1.  We need add a parameter to attach_volume(...) which indicates whether use
    cache or not. If yes, add the volume to cache group after it's attached.
2.  Some cache modules (bcache, dm-cache and so on) should be added into drivers.
3.  Cinder should setup a database record indicates which volume is cached.
4.  When detach a volume, we should remove the cache first.
5.  To support dynamically cache scheme, add and remove disk freely, we need
    organize the cached volume as a group. For bcache, backing devices can be
    attached and detached at runtime. We can also add bcache under same cache
    group interface for uniformity.

Since we have already implemented the grouping functionality for flashcache,
called flashcachegroup, abbreviated as fcg, as an example to illustrate the
procedure. Please refer to https://github.com/lihuiba/flashcachegroup for detail.

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

Add config option and relevant DB in Nova
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
