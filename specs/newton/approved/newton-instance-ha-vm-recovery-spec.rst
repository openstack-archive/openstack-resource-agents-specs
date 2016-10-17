..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
VM Recovery
==========================================

The purpose of this spec is to describe a method to recover
individual virtual machines that are marked as failed by
the VM monitoring method.

Problem description
===================
VM failure can be detected by VM monitoring method discussed in
`vm monitoring spec`__.

__ https://review.openstack.org/#/c/352217/

When VM failure event detected, must take the appropriate recovery
action to recover the VM. Those recovery actions are selected upon
the shared disk state, status of the VM,  and cause of the VM
failure and .etc.. These recovery actions should be configurable.
This spec is to describe what are the appropriate
actions to take for each cause of failures.


Use Cases
---------

As a cloud operator, I would like to provide my users with highly
available VMs to meet high SLA requirements. There are several types
of VM failure events that can occur in OpenStack clouds.
We need to make sure such events can be detected and recovered
by the system. Possible VM failure events include:

- VM Crashes.

- VM Hangs.

Possible recovery methods include:

- VM restart (stop and start)

- VM restart on different host

- Migrate VM (Live/Cold)

If a VM crashes, first approach to recovery is stop and start the
VM from nova-api.
Maximum restart threshold should be configurable and it could be
0, which means do not restart and go to next recovery method.
If restart fail or threshold is 0, it should try to restart VM
on a different host.


If a VM Hangs due to I/O error, the recovery service should disable
the ``nova-compute`` service on that host and restart the VM on a
different host. It could also migrate other VMs from the host, in
order to pre-empt an further I/O errors.


Proposed change
===============

VM monitors send failure events to recovery workflow service.
This workflow service can analyze the content of the failure event message
and execute the appropriate recovery action. This workflow service could also
handle the advanced recovery options such as maximum restart threshold,
execute next recovery action or execute multiple workflows.

Alternatives
------------

There are three alternatives to the proposed change:

1. Use Masakari as recovery workflow service

   VM monitors send the failure events to Masakari using Masakari
   notification API. Masakari will execute pre defined recovery actions.

2. User Mistral as recovery workflow service

   VM monitors call the Mistral workflow to execute execute appropriate
   recovery actions.

3. User Masakari as recovery engine and Mistral as workflow service

   VM monitors send the failure events to Masakari and Masakari will
   analyze the content of the failure event message and call Mistral
   workflow to execute recovery actions.


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



Developer impact
----------------


Implementation
==============

WIP


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




History
=======
