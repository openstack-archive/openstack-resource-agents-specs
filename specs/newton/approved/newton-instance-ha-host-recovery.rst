..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============
Host Recovery
=============

The purpose of this spec is to describe a method to recover all virtual
machines that are on the host after its failure.

Problem description
===================

In case of whole compute node failure, recovering of instances is crucial for
providing the high availability for the virtual machines. On the other hand,
automatic recovery of some instances may cause even more problems than the fact,
that they were suddenly turned off.

When taking both arguments into account it seems obvious that there is a need
for automatic recovery that has to be configurable, on both instance and host
level. This spec is to describe what are possible actions in case of compute
node failure and to describe the configuration. Automatic recovery of
particular instances is out of scope of this spec and would be described in
another document.

Use Cases
---------

* As a cloud operator, I would like to provide my users with highly
available VMs to meet high SLA requirements. Therefore, I need some of my VMs
to automatically resurrect after compute node failure.

Proposed change
===============

VMs recovery can be perform on the control plane of OpenStack cloud. It would be
done using mistral workflow service and pacemaker resource agent. The resource
agent would be responsible for starting the workflow, whereas mistral would
be responsible for performing *nova_evacuate* for each VM and for observing the
state of each evacuated VM. Usage of mistral would ensure that evacuation
workflow will end, even if some of the controllers dies during the process.

Alternatives
------------

1. We may not use mistral workflow at all and do all *nova_evacuate* related
stuff in the pacemaker resource agents. But this means that we would have to
implement all the HA mechanism in it, which would be difficult.

2. We may try to implement real *host-evacuate* in nova. Right now
*host-evacuate* iterate over all instances from given host on the client side.
We can try to change it and implement it in nova, but nova cores were against
this change in the past.

Data model impact
-----------------

None

API impact
----------

None

Security impact
---------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

There would be extra amount of RAM and CPU needed on each controller node to
run both pacemaker and mistral services. If they are already present on the
control plane, there would be no performance impact.

Other deployer impact
---------------------

Distributions need to package and deploy an extra services on each
controller node. Those services are mistral service and pacemaker resource
agent.

Developer impact
----------------

Nothing other than the listed work items below.

Implementation
==============

Resource agent would receive information from host monitor, that given host
is down. Then it would send a request to mistral to start recovery workflow.
Request needs to have below input parameters:

.. code-block:: json
    {
        "search_opts": {
            "host": COMPUTE_NAME
        },
        "on_shared_storage": [true|false]
    }

Assignee(s)
-----------

Primary assignee:
  <launchpad-id or None>

Other contributors:
  <launchpad-id or None>

Work Items
----------

* Prepare resource agent that would trigger mistral
* Prepare mistral workflow
* Document changes in HA guide

Dependencies
============

Host monitor

Testing
=======

Documentation Impact
====================

The service should be documented in the ha-guide.

References
==========

- `Instance HA etherpad started at Newton Design Summit in Austin
  <https://etherpad.openstack.org/p/newton-instance-ha>`_

- `"High Availability for Virtual Machines" user story
  <https://specs.openstack.org/openstack/openstack-user-stories/user-stories/proposed/ha_vm.html>`_

- `video of "HA for Pets and Hypervisors" presentation at OpenStack conference in Austin
  <https://youtu.be/lddtWUP_IKQ>`_

- `automatic-evacuation etherpad
  <https://etherpad.openstack.org/p/automatic-evacuation>`_

- `Instance auto-evacuation cross project spec (WIP)
  <https://review.openstack.org/#/c/257809>`_


History
=======
