..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
Add DM-snapshot for nova to Fast Boot Many Homogeneous Virtual Machines
=========================================================================

https://blueprints.launchpad.net/nova/+spec/thunder-boost

Nova supports to boot virtual machines (VMs) atop the Cinder volumes. However,
in the current implementation (version Icehouse), booting up a large number of
homogeneous VMs is time-consuming. To overcome this drawback, we propose a
lightweight patch called VMThunder for Nova for fast booting homogeneous VMs.
VMThunder accelerates the booting process of a large number of VMs through
on-demand data transfer.

Problem description
===================

Currently, Openstack provides three categories of methods for booting a virtual
machine: (i) booting from a local image or (ii) booting from a remote Cinder
volume or (iii) booting from snapshot.

* The first category needs to copy the entire image to a compute node, making it
suffer from a long transfer delay for large images.
* The second category remotely attaches a volume to a VM and transfers only the
necessary data from the volume, thus having better performance. However, this
approach only allows booting a single VM from a volume at a time. Moreover,
preparing a volume for each VM requires a long time.
* The third category is faster, but when launch multiple homogeneous vms on the
same host, public data will transfer many times from original volume, which
causes a lot of bandwidth-wasting.

As a result, it is currently inevitable to take a long time for booting a large
number of homogeneous VMs in Openstack.

Use Cases
----------
deployer impact:

We propose some modifications called VMThunder to address problems described above.
To use VMThunder, configure "nova.conf" set "use_vmthunder = true" and choose
"boot from volume" in the drop-down list.

VMThunder supports two kinds of approaches to create snapshot:

(i) VMM-snapshot: This is the current approach of OpenStack and deplorers can
  use this approach without any change.

(ii)  VMT-snapshot: This is VMThunder's approach to creating snapshot for fast
  booting, which uses the device mapper module to create a snapshot upon two
  volumes. One volume contains a base image with cache (snapshot_origin) for
  on-demand data transfer. The other volume (diff volume) is used to store image
  data different from the base image.

If you want to specify how to create snapshot, you should set
"snapshot=VMM" or "snapshot=DM".

Project Priority
-----------------
undefined

Proposed change
===============

We would like make full use of compute node's local storage, since compute
nodes of our cluster usually have SSD and spare storage space. Storage of
compute node can be used as cache for virtual machine image. Thus, the most
frequently used image data can be stored locally. We propose a mechanism that
adds a cache layer between original volume and the snapshot.

````
origin --> cache --> snapshot_1
            |
            |------> snapshot_2
            |
            |------> snapshot_3
````

VMThunder accelerates the deployment of vms in the following steps.

* Each compute node attaches the remote (original) image volume as the read-only
volume (VolumeO).
* Use local storage of each node as a cache of the read_only volume.
* Create a writable diff volume (VolumeU) to store data written by the VM.
* Make a snapshot to the cached volume and writable volume, and boot vm atop the
snapshot.

The created snapshot structure is depicted in the following figure.

````
+-----------------------------------------+
|             Snapshot                    |
+----------------------------+--+---------+
+----------------------------+  +---------+
|           Cache            |  | VolumeU |
+----------------------------+  +---------+
+----------------------------+
|          VolumeO           |
+----------------------------+

````

Besides operations of VMThunder described above, Our modification to Original
code of nova is light-weighted. Two major functions, i.e., the creation and
deletion of the original and diff volumes, are implemented as following:
(i) creation: We add a volume-driver class extends the original class
"DriverVolumeBlockDevice" in file "nova/virt/block_device.py" to prepare the
original volume and diff volume.
(ii) deletion: We add a delete method (about 20 lines) in file
"nova/compute/manager.py' to destroy the unused original and diff volumes.

More details of the implementation can be found in the following links:
Paper, http://www.computer.org/csdl/trans/td/preprint/06719385.pdf
Modification diff file, http://www.kylinx.com/vmthunder/diff2.txt

Alternatives
------------
(1) Backend storage optimization:
The distributed storages like NFS, cluster FS, distributed FS or SAN can
decrease the size of transferred volumes. However, the I/O pressure on the
storage servers increases dramatically when powering on a large number of
homogeneous VMs, since there may not be enough replicas on the storage servers
for offloading the I/O demands.

(2) Direct image access:
(https://blueprints.launchpad.net/nova/+spec/nova-image-zero-copy).
This approach uses the direct_url of the Glance v2 API, such that the number of
needed hops to transfer an image to a Nova-compute node is decreased. When
images are stored at multiple backend locations, the Nova-compute servers can
select a proper image storage for speeding up the downloading process.


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

We will significantly decrease the delay of booting up large numbers of
Cinder-volume-based VMs.

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

Primary assignee: VMThunderGroup (VMThunder)

Work Items
----------
* Add VMThunder package to create/delete VolumeO and VolumeU code
* Add new create/delete operations in nova
* Test with Nova (where most of this change really has an effect)

Dependencies
============
(1) Image cache:
Nova's image-caching facility reduces the start-up time for creating
homogeneous virtual machines on one nova-compute node. However, it helps
neither the first-time provisioning nor the Cinder-based booting process.

(2) Multi-attach volume:
(https://wiki.openstack.org/wiki/Cinder/blueprints/multi-attach-volume)
This approach allows a volume to be attached to more than one instance
simultaneously. As a result, volumes can be shared among multiple guests when
the instances are already available. Besides, these volumes can also be used
for booting a number of VMs by enforcing the multi-attach volumes as read-only
image disks.

Testing
=======
in order to show the effectiveness we will add necessary tests into nova's test
framework.
*add unit tests
*have CI running tempest for Kilo, which will validate this work.

Documentation Impact
====================

We need to document how to create many homogeneous virtual machines though our
new option.

References
==========

VMThunder: http://vmthunder.github.io/

Mailing list:
http://lists.openstack.org/pipermail/openstack-dev/2014-April/032883.html

VMThunder Publication:http://vmthunder.github.io/blog/2014/03/02/publication/
