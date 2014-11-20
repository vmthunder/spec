..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cache support for Cinder
===============================================================================
  
https://blueprints.launchpad.net/cinder/+spec/

HDD as conventional storage device cannot provide adequate performance compared
to fast evolving CPU, memory and network. HDD has become a performance
bottleneck of overall system. High-end storage(SSD, even memory) is much
faster than HDD, but has low capacity, short lifetime and high price issues.
Some block-level caching solutions(flashcache, bcache, dm-cache, lvm-cache)
enhance storage performance by using a fast device as cache of the slow devices.


Problem description
===================

Currently, there is no cache support for block device in cinder. Data is stored
in cinder servers and attached to compute nodes. To take full advantage of HDDs
and SSDs of compute nodes, we need to cache data in them. Thus, compute nodes
will access data in local cache before send requests to servers. To do so, there
are some challenges.

1.  Since compute nodes dynamically attach and release volumes from servers,
the cache scheme must support dynamically changing configurations, can add and
remove disks freely.
2.  The cache scheme for cinder should support multiple cache modules(such as
flashcache, bcache, dm-cache, lvm-cache).


Proposed change
===============

On the CinderClient side, we should set up a config option to indicate whether
use SSD Cache or not. If this option is true, SSD Cache environment will be
initiated when CinderClient start. When Nova use/attach to a volume in Cinder,
Nova should pass a parameter to indicate whether this volume will be cached in 
computer node or not. CinderClient add a volume to the SSD Cache according to 
Nova's parameter. Cinder should set up a DB to record which volume is cached.
The main code to implement cache scheme can be put to the path /cinder/volume or
/cinder/volume/driver in Cinder, CinderClient use it through RPC. We have
already implement FlashCacheGroup Python Package to make cache of a group of
HDDs by using one or multiple SSDs freely, so that steps can be implemented by
flash cache theoretically, but bcache is another good choice.


Alternatives
------------

DM-Cache
DM-Cache uses I/O scheduling and cache management techniques optimized for
flash-based SSDs. The device mapper target (DM-Cache) reuses the metadata
library used in the thin-provisioning library. Both write-back and
write-through are supported by DM-Cache. But DM-Cache's metadata device is
not easy to handle, and SSD still can not be removed freely.

LVM-Cache
LVM-Cache is built on top of DM-Cache so that logical volumes can be turned into
cache devices. Because of that, LVM-Cache splits the cache pool LV into two
devices - the cache data LV and cache metadata LV. LVM-Cache will face the some
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
None

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

Add config option and api in CinderClient
Add main code and relevant DB in Cinder
Modify relevant api in Nova
Add unit tests


Dependencies
============

For using flash cache, flashcachegroup Python package and facebook's flashcache
must already installed in running environment, other dependencies will upon
specific case.

Testing
=======

The unit tests and integrated tests will be added to the component.

Documentation Impact
====================
None


References
==========

Flashcachegroup: https://github.com/lihuiba/flashcachegroup

