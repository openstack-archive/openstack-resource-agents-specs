..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VM Monitoring
==========================================

The purpose of this spec is to address a method for monitoring the
health of OpenStack VM instances without access to the VMs' internals.

Problem description
===================

Monitoring VM health is essential for providing high availability for
the VMs. Typically cloud operators cannot access inside VMs in order
to monitor their health, because this would violate the contract
between cloud operators and users that users have complete autonomy
over the contents of their VMs and all actions are performed inside
them. Operators cannot assume any knowledge of the software stack
inside the VM or make any changes to it. Therefore, VM health
monitoring must be done externally. This VM monitor must be able to
detect VM crashes, hangs (e.g. due to I/O errors) and so on.

Use Cases
---------

As a cloud operator, I would like to provide my users with highly
available VMs to meet high SLA requirements. Therefore, I need my VMs
automatically monitored for sudden stops, crashes, IO failures and
similar.  Any VM failure event detected needs to be passed to a VM
recovery workflow service which takes the appropriate actions to
recover the VM.  For example:

- If a VM crashes, the recovery service will try to restart it,
  possibly on the same host at first, and then on a different host if
  it fails to restart or if it restarts successfully but then crashes
  a second time on the original host.

- If a VM receives an I/O error, the recovery service may prefer to
  immediately contact ``nova-api`` to centrally disable the
  ``nova-compute`` service on that host (so that no new VMs are
  scheduled on the host) and restart the VM on a different host. It
  could also potentially live-migrate all other VMs off that host, in
  order to pre-empt an further I/O errors.

Proposed change
===============

VM monitoring can be done at the hypervisor level without accessing
inside the VMs.  In particular, |libvirt|_ provides a mechanism for
monitoring its event stream via an event loop.  We need to filter the
required events and pass them to a recovery workflow service.  In
order to eliminate redundancy and improve extensibility, these event
filters must be configurable.

.. |libvirt| replace:: `libvirt`
.. _libvirt: https://libvirt.org/

Potential advantages:

- Catching events at their source (the hypervisor layer) means that we
  don't have to rely on ``nova`` having knowledge of those events.
  For example, ``libvirtd`` can output errors when a VM's I/O layer
  encounters issues, but ``nova`` doesn't emit corresponding events for
  this.
- It should be relatively easy to support a configurable event filter.
- The VM instance monitor can be run on each compute node, so it should
  scale well as the number of compute nodes increases.
- The VM instance monitors could be managed by `pacemaker_remote`__ via a
  new `OCF RA (resource agent)`__.

__ http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Remote/
__ http://www.linux-ha.org/wiki/OCF_Resource_Agents

Alternatives
------------

There are three alternatives to the proposed change:

1. Listen for VM status change events on message queue.

   Potential disadvantages:

   - It might be less reliable, if for some reason the
     message queue introduced latency or was lossy.

   - There also might be some gaps in which events are propagated to
     the queue; if so, we could submit a ``nova`` spec to plug the gaps.

   - If we listen for events from the control plane, it won't scale as
     well to large numbers of compute nodes, and then would be awkward
     to trigger recovery via Pacemaker.

2. Write a new ``nova-libvirt`` OCF RA

   It would compare ``nova``'s expectations of which VMs should be running
   on the compute node with the reality.  Any differences between the
   two would send appropriate failure events to the recovery workflow
   service.

   Potential disadvantages:

   - This is more complexity than is expected to run inside an RA.
     RAs are supposed to be lightweight components which simply start,
     stop, and monitor services, whereas this would require abusing
     that model by pretending there is a separate monitoring service
     when there isn't. The ``monitor`` action would need to fail when
     any differences as mentioned above were detected, and then the
     ``stop`` or ``start`` action would need to send the failure
     events.

   - Within this "fake service" model, it's not clear how to avoid
     sending the same failure events over and over again until the
     failures were corrected.

   - Typically RAs are implemented in ``bash``.  This is not a hard
     requirement, but something of this complexity would be much
     better coded in Python, resulting in either a mix of languages
     within the `openstack-resource-agents`_ repository

3. Same as 2. above, but as part of the NovaCompute_ RA

   - This has all the disadvantages of 2., but even more so, since
     new functionality would have to be mixed alongside the existing
     NovaCompute_ functionality.

.. _openstack-resource-agents: https://launchpad.net/openstack-resource-agents
.. _NovaCompute: https://github.com/openstack/openstack-resource-agents/blob/master/ocf/NovaCompute

Data model impact
-----------------

None

API impact
----------

The HTTP API of the VM recovery workflow service needs to be able to
receive events in the format they are sent by this instance monitor.

Security impact
---------------

Ideally it should be possible for the instance monitor to send
instance event data securely to the recovery workflow service
(e.g. via TLS), without relying on the security of the admin network
over which the data is sent.

Other end user impact
---------------------

None

Performance Impact
------------------

There will be a small amount of extra RAM and CPU required on each
compute node for running the instance monitor.  However it's a
relatively simple service, so this should not have significant impact
on the node.

Other deployer impact
---------------------

Distributions need to package and deploy an extra service on each
compute node.  However the existing `instance monitor`_ implementation
in masakari_ already provides files to simplify packaging on the Linux
distributions most commonly used for OpenStack infrastructure.

.. _masakari: https://github.com/ntt-sic/masakari
.. _`instance monitor`:
   https://github.com/ntt-sic/masakari/tree/master/masakari-instancemonitor/

Developer impact
----------------

Nothing other than the listed work items below.

Implementation
==============

``libvirtd`` uses `QMP (QEMU Machine Protocol)`__ via UNIX domain
socket (``/var/lib/libvirt/qemu/xxxx.monitor``) to communicate with
the VM domain.  ``libvirt`` catches the failure events and passes them
to the VM monitor.  The VM monitor filters the events and passes them
to an external recovery workflow via HTTP, which then takes the action
required to recover the VM.

__ http://wiki.qemu.org/QMP

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

We can almost certainly reuse the `instance monitor`_ provided
by masakari_.

**FIXME**:

- Need to detail how and in which format the event data should
  be sent over HTTP.  **This should allow for support for other
  hypervisors not based on** ``libvirt`` **being added in the future.**
- Need to give details of in which exact ways the service can
  be configured.

  - How should event filtering be configurable?

  - Where should the configuration live?  With `masakari`, it
    lives in ``/etc/masakari-instancemonitor.conf``.

Assignee(s)
-----------

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

- Package `masakari`_'s `instance monitor`_ for SLES (`aspiers`)
- Add documentation to the |ha-guide|_ (`beekhof`)
- Look into libvirt-test-API_
- Write test suite

.. |ha-guide| replace:: OpenStack High Availability Guide
.. _ha-guide: http://docs.openstack.org/ha-guide/
.. _libvirt-test-API: https://libvirt.org/testapi.html

Dependencies
============

- `libvirt <https://libvirt.org/>`_
- `libvirt's Python bindings <https://libvirt.org/python.html>`_

Testing
=======

It may be possible to write a test suite using libvirt-test-API_ or
at least some of its components.

Documentation Impact
====================

The service should be documented in the |ha-guide|_.

References
==========

- `Instance HA etherpad started at Newton Design Summit in Austin
  <https://etherpad.openstack.org/p/newton-instance-ha>`_

- `"High Availability for Virtual Machines" user story
  <http://specs.openstack.org/openstack/openstack-user-stories/user-stories/proposed/ha_vm.html>`_

- `video of "HA for Pets and Hypervisors" presentation at OpenStack conference in Austin
  <https://youtu.be/lddtWUP_IKQ>`_

- `automatic-evacuation etherpad
  <https://etherpad.openstack.org/p/automatic-evacuation>`_

- `Instance auto-evacuation cross project spec (WIP)
  <https://review.openstack.org/#/c/257809>`_


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Newton
     - Introduced
