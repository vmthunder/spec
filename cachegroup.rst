..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cachegroup support
===============================================================================

https://blueprints.launchpad.net/nova/+spec/

Traditional hard disk drive (HDD) has become a performance bottleneck compared
with CPU, memory and network. High-end storage (solid state drive, even memory)
is much faster than HDD, but has lower capacity, shorter lifetime and higher
price issues. Some block-level caching solutions (flashcache, bcache, dm-cache,
lvm-cache) enhance storage performance by using a fast device as cache of the
slow devices. We propose to improve these solutions by enabling a group of fast
caching devices to collaboratively service a group of slow storage devices.


Problem description
===================

Currently, there is no general cache support for block device in Nova. Data is
stored in volumes of cinder servers and attached to compute nodes. To take full
advantage of local storage devices (HDDs and SSDs) of the compute nodes, we
need to cache data on them. Thus, compute nodes will access data in local
cache before send I/O requests to servers. To achieve this goal, there are some
challenges.

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
We propose the following changes:

1.  Nova should set up a configuration option to indicate whether
    to use the cache or not. If this option is True, the cache environment
    will be initiated when the Nova starts.
2.  When Nova uses/attaches to a volume in Cinder, Nova should pass a
    parameter to indicate whether this volume will be cached in compute node
    or not. If cache wanted, this volume will be added to local cache before
    a instance connect to it.
3.  When Nova detaches to a volume in Cinder, Nova should checkout this
    volume whether it is be cached or not. If this volume has already been
    cached, Nova should wait it be removed from local cache and then start
    general detach steps.
4.  Nova should set up a database record indicates which volume is cached.
5.  Dynamically make cache for a group of (one or multiple) HDDs by using a
    group of SSDs, the details of which is as follows. (Since we have already
    implemented the grouping functionality for flashcache, we take
    FlashCacheGroup, abbreviated as fcg, as an example to illustrate the
    procedure.)

*  Fcg uses dm-linear to create a logical group of HDDs and combine all SSDs.
*  Fcg makes cache of logical HDD group with the linear combined SSDs,
   called cached group.
*  When adding HDD to the logical HDD group, fcg splits a cached HDD out of
   the cached group by using dm-linear accordingly.
*  When removing HDD from the logical HDD group, fcg also removes the cached
   HDD accordingly.

(refer to https://github.com/lihuiba/flashcachegroup for detail)

Besides fcg, we are also implementing the cache group functionality for
bcache following the same procedure.


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
