..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VM Recovery
==========================================

The purpose of this spec is to describe a method to recover
individual virtual machines that are marked as failed by
the VM monitoring component.

Problem description
===================

VM failure can be detected by VM monitoring method discussed in
`vm monitoring spec`__.

__ https://review.openstack.org/#/c/352217/

When VM failure event is detected, appropriate recovery actions must
be taken. Those recovery actions should be decided using configurable
policies based on inputs such as the state of storage (shared or
otherwise), status of the VM, and cause of the VM failure.

Use Cases
---------

As a cloud operator, I would like to provide my users with highly
available VMs to meet high SLA requirements. There are several types
of VM failure events that can occur in OpenStack clouds.
We need to make sure such events can be detected and recovered
by the system. Possible VM failure events include:

- VM crashes

- VM hangs

Possible recovery methods include:

- VM restart (stop and start)

- VM restart on different host

Scope
-----

This spec only addresses recovery from isolated failures of individual
VMs.  Monitoring of the VMs, and detection and recovery from wider
failures, such as failure of a whole compute host, will be covered by
separate specs, and are therefore out of scope for this spec.

This spec has the following goals:

1. Encourage all implementations of VM recovery, whether upstream or
   downstream, to receive failure notifications in a standardized
   manner.  This will allow cloud vendors and operators to implement
   HA of the compute plane via a collection of compatible components
   (of which one is compute node monitoring), whilst not being tied to
   any one implementation.

2. Suggest appropriate actions which can be taken for each failure
   case.

3. Provide details of and recommend a specific implementation which
   for the most part already exists and is proven to work.

4. Identify gaps with that implementation and corresponding future
   work required.

Proposed change
===============

VM monitors send failure events to a recovery workflow service.  This
workflow service can analyze the content of the failure event message
and execute the appropriate recovery action. This workflow service
could also handle the advanced recovery options such as maximum
restart threshold, execute next recovery action or execute multiple
workflows.

If a VM crashes, the first approach to recovery is stop and start the
VM from nova-api.  The maximum restart threshold should be
configurable, and it could be 0, which means do not restart and go to
next recovery method.  If restart fails, or threshold is 0, it should
try to restart the VM on a different host.

If a VM hangs due to an I/O error, the recovery service may be
required to automatically disable the ``nova-compute`` service on that
host and restart the VM on a different host. It could also migrate
other VMs from the host, in order to preempt further I/O errors.

Implementation
==============

There are at least three possible ways to implement the proposed
change:

1. Use Masakari as recovery workflow service

   VM monitors send the failure events to Masakari using Masakari's
   notification API. Masakari will execute pre-defined recovery actions.

2. Use Mistral as recovery workflow service

   VM monitors call the Mistral workflow to execute execute appropriate
   recovery actions.

3. Use Masakari as recovery engine and Mistral as workflow service

   VM monitors send the failure events to Masakari and Masakari will
   analyze the content of the failure event message and call Mistral
   workflow to execute recovery actions.


Data model impact
-----------------

None

REST API impact
---------------

The HTTP API of the VM recovery workflow service needs to be able to
receive events in the format they are sent by the VM monitor.

Security impact
---------------

Ideally it should be possible for the VM monitor to send instance
event data securely to the recovery workflow service (e.g. via TLS),
without relying on the security of the admin network over which the
data is sent.

Other end user impact
---------------------

None

Performance Impact
------------------

None

Other deployer impact
---------------------


Developer impact
----------------

Documentation Impact
--------------------

The service should be documented in the |ha-guide|_.

.. |ha-guide| replace:: OpenStack High Availability Guide
.. _ha-guide: http://docs.openstack.org/ha-guide/

Assignee(s)
-----------

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>


Work Items
==========

 WIP

Dependencies
============


Testing
=======


Documentation Impact
====================



References
==========



History
=======
