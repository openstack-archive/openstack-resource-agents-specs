..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VM Monitoring
==========================================
The purpose of this spec is to address a method for monitor the health of the
VMs without accessing to the VMs.

Problem description
===================
Monitoring the VM health is essential for provide high availability for the
VMs. Most of the cases, operator can not access to VM for monitor the
the health of VM. Therefore, VM health monitoring must be done without
accessing to the VM. This VM monitor must be able to detect VM crash,
hangs (e.g. due to I/O errors)..etc..


Use Cases
---------
As a cloud operator, I would like to provide my users with highly available VMs
to meet high SLA requirements. Therefore, I need to monitor my highly available
VMs for sudden stops, crashes, IO failures..etc.
Then, those VM failure events needs to be passed to VM recovery workflow,
to take the appropriate actions to recover the VM.


Proposed change
===============
VM monitoring can be done from libvirt level without accessing to the VMs.
Need to filter the required events and pass them to recovery workflow.
In order to eliminate redundancy and improve the extensibility,
these event filters must be configurable.

(WIP) Need to choose one from the alternatives to go with.


Alternatives
------------
Following 4 options are available:

* listen for VM status change events on MQ

might be some gaps here; if so, we could submit a nova spec
but if we do it from the control plane, it won't scale, and then would be
awkward to trigger recovery via Pacemaker.

* monitor via libvirt event loop, like masakari already does:

https://github.com/ntt-sic/masakari/blob/master/masakari-instancemonitor/instancemonitor/masakari_instancemonitor.py
this should be more reliable than listening to MQ since doesn't
depend on local MQ
Can also catch extra events which nova doesn't emit (I/O errors)
should support configurable event filter
instance monitor could be managed by OCF RA

* new nova-libvirt OCF RA:

It would compare nova's expectations of which VMs should be running on the
compute node with the reality any differences between expectations and
reality trigger recovery somehow

* same as above, but as part of NovaCompute RA



Data model impact
-----------------

None


REST API impact
---------------

None

Security impact
---------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None


Implementation
==============

(WIP) if we choose to go with monitor via libvirt event loop

libvirtd use QMP (Qemu Machine Protocol) via domain socket
(/var/lib/libvirt/qemu/xxxx.monitor) to communicate with domain.
Then, libvirt catch the failure events and pass them to VM monitor.
VM monitor filter and pass them to recovery workflow to take
required action to recover the VM.

::

 +-----------------------+
 | +----------------+    |
 | |       VM       |    |
 | | (qemu Process) |    |
 | +---------^------+    |
 |       |   |QMP        |
 | +-----v----------+    |
 | |    libvirtd    |    |
 | +---------^------+    |
 |       |   |           |
 | +-----v----------+    |        +-----------------------+
 | |    VM Monitor  +------------>+  VM recovery workflow |
 | +----------------+    |        +-----------------------+
 |                       |
 | Compute Node          |
 +-----------------------+





Assignee(s)
-----------

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------


Dependencies
============


Testing
=======




Documentation Impact
====================



References
==========

Newton summit Instance HA etherpad
https://etherpad.openstack.org/p/newton-instance-ha



History
=======


.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
