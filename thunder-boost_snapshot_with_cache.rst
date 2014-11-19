..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================================================
A Lightweight Proposal for Fast Booting Many Homogeneous Virtual Machines
=========================================================================

https://blueprints.launchpad.net/nova/+spec/thunder-boost

Nova supports to boot virtual machines (VMs) atop the Cinder volumes. However,
in the current implementation (version j), booting up a large number of
homogeneous VMs is time-consuming. To overcome this drawback, we propose a
lightweight patch called vmthunder for Nova for fast booting homogeneous VMs. 
vmthunder accelerates the booting process through on-demand data transfer.

Problem description
===================

Currently, Openstack provides three categories of methods for booting a virtual
machine: (i) booting from a local image or (ii) booting from a remote Cinder
volume or (iii)booting from snapshot.


- The first category needs to copy the entire image to a compute node, making it suffer from a long transfer delay for large images. 

- The second category remotely attaches a volume to a VM and transfers only the necessary data from the volume, thus having better performance. However, this approach only allows booting a single VM from a volume at a time. Moreover, preparing a volume for each VM requires a long time.

- The third category is faster, but when launch multiple homogeneous vms on the same host, public data will transfer many times from original volume which causes a lot of bandwidth-wasting. 

As a result, it is currently inevitable to take a long time for booting a large number of homogeneous VMs in Openstack. 

Use Cases
----------
Deployer impact: for users who want to fast boot multiple homogeneous VMs,
config "nova.conf" set "use_vmthunder = true" and choose "boot from volume",
then you can launch a large number of VMs within 2 minutes. There is also an
optional choice to indicate how to create a snapshot, vmthunder support two kinds of snapshot:

- VMM-snapshot: using VMM to create a snapshot, which supports all kinds of virtual machine technology like Xen, Virtual BOX, VMWare etc.
- DM-snapshot: using device mapper module to create a snapshot upon two volumes. One volume contains a base image with cache(snapshot_origin) for on-demand data transfer. The other volume(diff volume) is used to store diff-image.Defaultly, vmthunder use DM-snapshot to create a snapshot.

Project Priority
-----------------
undefined

Proposed change
===============

We propose some modifications called vmthunder to address problems described above. Vmthunder accelerate the deployment of vms in following ways.

1. Each compute node attaches image volume to local as original volume.
2. Use local storage of each node as a cache of the original volume.
3. Make snapshot to the cached volume, boot vm atop the snapshot.

At the stage "_prep_block_device()", vmthunder configures each VM with
two volumes(shown above):a (read-only) template volume exactly the same as the
pre-created original volume and a (writable)snapshot volume storing each VM's
difference to the template. The original volume is the root of a template
volume relay tree, and each VM fetches only the necessary data from its parent
over the multi-path iSCSI protocol. In addition, vmthunder makes use of a
compute node's local storage as a cache to accelerate the image transferring
process and avoid a repetitive data transfer. On-demand data transfer
dramatically accelerates VMs' booting process.

The following diagram shows the architecture of image carrier.

````

                   +-------------------------------------+
                   |              Original               |
                   |               Volume                |
                   +-------------------------------------+

+-+----+-------+-+  +-+-------+----+-+  +-+----+-++----+-+  +-+----+--+----+-+
| +----+  +----+ |  | +----+  +----+ |  | +----+  +----+ |  | +----+  +----+ |
| | TV |  | SV | |  | | TV |  | SV | |  | | TV |  | SV | |  | | TV |  | SV | |
| +----+  +----+ |  | +----+  +----+ |  | +----+  +----+ |  | +----+  +----+ |
+-+----+--+----+-+  +-+----+--+----+-+  +-+----+--+----+-+  +-+----+--+----+-+


                    +-----------------+-------------+
                    |         Block Device          |
                    |                               |
                    | +--------------------------+  |
                    | |   TV:Template Volume     |  |
                    | |   (Base image, mounted   |  |
                    | |   from parent Via iSCSI) |  |
                    | +--------------------------+  |
                    | +--------------------------+  |
                    | |   SV: Snapshot Volume    |  |
                    | |   (Diff image)           |  |
                    | |                          |  |
                    | +--------------------------+  |
                    |                               |
                    +-------------------------------+


````

Our modification to Nova itself is light-weighted (about 80 lines of insertions
and deletions). Two major functions, i.e., the creation and deletion of the
template and snapshot volumes, are implemented as following: 
(i) creation: We add a volume-driver class extends the original class 
"DriverVolumeBlockDevice" in file "nova/virt/block_device.py" to prepare the
template and snapshot volumes. 
(ii) deletion: We add a delete method (about 20 lines) in file
"nova/compute/manager.py' to destroy the unused template and snapshot volumes.

More details of the implementation can be found in the following links:
Paper, http://www.computer.org/csdl/trans/td/preprint/06719385.pdf
Modification diff file, http://www.kylinx.com/vmthunder/diff2.txt

Alternatives
------------
(1)Backend storage optimization:
The distributed storages like NFS, cluster FS, distributed FS or SAN can
decrease the size of transferred volumes. However, the I/O pressure on the
storage servers increases dramatically when powering on a large number of
homogeneous VMs, since there may not be enough replicas on the storage servers
for offloading the I/O demands.

(2)Direct image access:
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

Primary assignee: vmThunderGroup (vmthunder)

Work Items
----------
* Add vmthunder package to create/delete TV and SV code	 
* Add new create/delete operations in nova
* Test with Nova (where most of this change really has an effect)

Dependencies
============
(1)Image cache:
(https://blueprints.launchpad.net/cinder/+spec/add-flashcachegroup-support)
Nova's image-caching facility reduces the start-up time for creating
homogeneous virtual machines on one nova-compute node. However, it helps
neither the first-time provisioning nor the Cinder-based booting process.

(2)Multi-attach volume:
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
