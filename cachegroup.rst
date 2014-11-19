..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================================================================
Add cachegroup support
===============================================================================
  
https://blueprints.launchpad.net/cinder/+spec/

Block devices stored on tranditional stroage device like HDD still can't afford
enough speed of aceess and become the performance bottleneck of overall system. 
With availability of economical flash storage technology, block-level caching
solutions become a acceptable choice to improve stroage performance.Some
block-level caching solutions(falsh cache, bcache, dm-cache, lvm-cache) allows
one to use a fast block device such as SSD as cache to accelerate a slower
drive. 


Problem description
===================

Currently, there is not a common data cache mechanism for block device storage
in Cinder. The block device on the compute nodes are dynamically mounted from
the storage server, therefore Cinder requires a dynamically configurable cache
tool. Using cache mechanism on the compute nodes can improve the performance of
compute nodes and ease the pressure of the storage server. Flashcachegroup
enables a group of disk(s) (on storage server) shares a cache mechanism
supplied by one or multiple SSDs (on compute nodes) to save read/write time.
For the purpose of easy management, simply creating a group of hard disks is
also feasible. Hard disk(s) can be dynamically added to or removed from the
group on demand.

Proposed change
===============

On the CinderClient side, we should set up a config option to indicate whether
use SSD Cache or not. If this option is true, SSD Cache environment will be
initiated when CinderClient start. When Nova use/attach to a volume in Cinder,
Nova should pass a parameter to indicate whether this volume will be cached in 
computer node or not. CinderClient add a volume to the SSD Cache according to 
Nova's parameter. Cinder should set up a DB to record which volume is cached. 

We recommend to use FlashCacheGroup to implement the steps above.The
implementation goal of fcg is to makes cache of a group of HDDs by using
one or multiple SSDs.
Fcg achieves its goal through following steps:
1. Fcg uses dm-linear to create a logical group of HDDs and combine all SSDs.
2. Fcg makes cache of logical HDD group with the linear combined SSDs, 
called cached group.
3. When adding HDD to the logical HDD group, fcg splits a cached HDD out of
cached group by using dm-linear accordingly.
4. When removing HDD from the logical HDD group, fcg also removes the cached
HDD accordingly.

Currently, FlashCacheGroup support flash cache only, but may support other
SSD caching module like bcache later.



Alternatives
------------

DM-Cache
DM-Cache uses I/O scheduling and cache management techniques optimized for
flash-based SSDs. The device mapper target (DM-Cache) reuses the metadata
library used in the thin-provisioning library. Both write-back and
write-through are supported by DM-Cache

LVM-Cache
LVM-Cache is built on top of DM-Cache so that logical volumes can be turned into
cache devices. Because of that, LVM-Cache splits the cache pool LV into two
devices - the cache data LV and cache metadata LV. 

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

Modify relevant api in Nova
Add cache setup environment in CinderClient
Add relevant DB in Cinder
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

