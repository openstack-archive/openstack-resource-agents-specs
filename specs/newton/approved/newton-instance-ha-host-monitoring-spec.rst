..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Host Monitoring
==========================================

The purpose of this spec is to describe a method for monitoring the
health of OpenStack compute nodes.

Problem description
===================

Monitoring compute node health is essential for providing high
availability for VMs. A health monitor must be able to detect crashes,
freezes, network connectivity issues, and any other OS-level errors on
the compute node which prevent it from being able to run the necessary
services in order to host existing or new VMs.

Use Cases
---------

As a cloud operator, I would like to provide my users with highly
available VMs to meet high SLA requirements. Therefore, I need my
compute nodes automatically monitored for hardware failure, kernel
crashes and hangs, and other failures at the operating system level.
Any failure event detected needs to be passed to a compute host
recovery workflow service which can then take the appropriate remedial
action.

For example, if a compute host fails (or appears to fail to the extent
that the monitor can detect), the recovery service will typically
identify all VMs which were running on this compute host, and may take
any of the following possible actions:

- Fence the host (STONITH) to eliminate the risk of a still-running
  instance being resurrected elsewhere (see the next step) and
  simultaneously running in two places as a result, which could cause
  data corruption.

- Resurrect some or all of the VMs on other compute hosts.

- Notify the cloud operator.

- Notify affected users.

- Make the failure and recovery events available to telemetry /
  auditing systems.

Scope
-----

This spec only addresses monitoring the health of the compute node
hardware and basic operating system functions, and notifying
appropriate recovery components in the case of any failure.

Monitoring the health of ``nova-compute`` and other processes it
depends on, such as ``libvirtd`` and anything else at or above the
hypervisor layer, including individual VMs, will be covered by
separate specs, and are therefore out of scope for this spec.

Any kind of recovery workflow is also out of scope and will be covered
by separate specs.

This spec has the following goals:

1. Encourage all implementations of compute node monitoring, whether
   upstream or downstream, to output failure notifications in a
   standardized manner.  This will allow cloud vendors and operators
   to implement HA of the compute plane via a collection of compatible
   components (of which one is compute node monitoring), whilst not
   being tied to any one implementation.

2. Provide details of and recommend a specific implementation which
   for the most part already exists and is proven to work.

3. Identify gaps with that implementation and corresponding future
   work required.

Acceptance criteria
===================

Here the words "must", "should" etc. are used with the strict meaning
defined in `RFC2119 <https://www.ietf.org/rfc/rfc2119.txt>`_.

- Compute nodes must be automatically monitored for hardware failure,
  kernel crashes and hangs, and other failures at the operating system
  level.

- The solution must scale to hundreds of compute hosts.

- Any failure event detected must cause the component responsible for
  alerting to send a notification to a configurable endpoint so that
  it can be consumed by the cloud operator's choice of compute node
  recovery workflow controller.

- If a failure notification is not accepted by the recovery component,
  it should be persisted within the monitoring/alerting components,
  and sending of the notification should be retried periodically until
  it succeeds.  This will ensure that remediation of failures is never
  dropped due to temporary failure or other unavailability of any
  component.

- The alerting component must be extensible in order to allow
  communication with multiple types of recovery workflow controller,
  via a driver abstraction layer, and drivers for each type.  At least
  one driver must be implemented initially.

- One of the drivers should send notifications to an HTTP endpoint
  using a standardized JSON format as the payload.

- Another driver should send notifications to the `masakari API server
  <https://wiki.openstack.org/wiki/Masakari#Masakari_API_Design>`_.

Implementation
==============

The implementation described here was presented at OpenStack Day
Israel, June 2017, from which `this diagram
<https://aspiers.github.io/openstack-day-israel-2017-compute-ha/#/no-fence_evacuate>`_
should assist in understanding the below description.

Running a `pacemaker_remote
<http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Remote/>`_
service on each compute host allows it to be monitored by a central
Pacemaker cluster via a straight-forward TCP connection.  This is an
ideal solution to this problem for the following reasons:

- Pacemaker can scale to handling a very large number of remote nodes.

- ``pacemaker_remote`` can be simultaneously used for monitoring and
  managing services on each compute host.

- ``pacemaker_remote`` is a very lightweight service which will not
  cause any significantly increased load on each compute host.

- Pacemaker has excellent support for fencing for a wide range of
  STONITH devices, and it is easy to extend support to other devices,
  as shown by the `fence_agents repository
  <https://github.com/ClusterLabs/fence-agents>`_.

- Pacemaker is easily extensible via OCF Resource Agents, which allow
  custom design of monitoring and of the automated reaction when those
  monitors fail.

- Many clouds will already be running one or more Pacemaker clusters
  on the control plane, as recommended by the |ha-guide|_, so
  deployment complexity is not significantly increased.

- This architecture is already implemented and proven via the
  commercially supported enterprise products RHEL OpenStack Platform
  and SUSE OpenStack Cloud, and via `masakari
  <https://github.com/openstack/masakari/blob/master/README.rst>`_
  which is used by production deployments at NTT.

Since many different tools are currently in use for deployment of
OpenStack with HA, configuration of Pacemaker is currently out of
scope for upstream projects, so the exact details will be left as the
responsibility of each individual deployer.  Nevertheless, examples
of partial configurations for Pacemaker are given below.

Fencing
-------

Fencing is technically outside the scope of this spec, in order to
allow any cloud operator to choose their own clustering technology
whilst remaining compliant and hence compatible with the notification
standard described here.  However, Pacemaker offers such a convenient
solution to fencing which is also used to send the failure
notification, so it is described here in full.

Pacemaker already implements effective heartbeat monitoring of its
remote nodes via the TCP connection with ``pacemaker_remote``, so it
only remains to ensure that the correct steps are taken when the
monitor detects failure:

1. Firstly, the compute host must be fenced via an appropriate STONITH
   agent, for the reasons stated above.

2. Once the host has been fenced, the monitor must mark the host as
   needing remediation in a manner which is persisted to disk (in case
   of changes in cluster state during handling of the failure) and
   read/write-accessible by a separate alerting component which can
   hand over responsibility of processing the failure to a recovery
   workflow controller, by sending it the appropriate notification.

These steps should be implemented by using two features of Pacemaker.
Firstly, its ``fencing_topology`` configuration directive to implement
the second step as a custom fencing agent which is triggered after the
first step is complete.  For example, the custom fencing agent might
be set up via a Pacemaker ``primitive`` resource such as:

.. code::

    primitive fence-nova stonith:fence_compute \
        params auth-url="http://cluster.my.cloud.com:5000/v3/" \
               domain=my.cloud.com \
               tenant-name=admin \
               endpoint-type=internalURL \
               login=admin \
               passwd=s3kr1t \
        op monitor interval=10m

and then it could be configured as the second device in the fencing
sequence:

.. code::

    fencing_topology compute1: stonith-compute1,fence-nova

Secondly, the ``fence_compute`` agent here should persist the marking of
the fenced compute host via `attrd
<http://clusterlabs.org/man/pacemaker/attrd_updater.8.html>`_, so that
a separate alerting component can transfer ownership of this host's
failure to a recovery workflow controller by sending it the
appropriate notification message.

It is worth noting that the ``fence_compute`` fencing agent `already
exists
<https://github.com/ClusterLabs/fence-agents/blob/master/fence/agents/compute/fence_compute.py>`_
as part of an earlier architecture, so it is strongly recommended to
reuse and adapt the existing implementation rather than writing a new
one from scratch.

Sending failure notifications to a host recovery workflow controller
--------------------------------------------------------------------

There must be a highly available service responsible for taking host
failures marked in ``attrd``, notifying a recovery workflow
controller, and updating ``attrd`` accordingly once appropriate action
has been taken.  A suggested name for this service is
``nova-host-alerter``.

It should be easy to ensure this alerter service is highly available
by placing it under management of the existing Pacemaker cluster.  It
could be written as an `OCF resource agent
<http://www.linux-ha.org/doc/dev-guides/ra-dev-guide.html>`_, or as a
Python daemon which is controlled by an OCF / LSB / ``systemd`` resource
agent.

The alerter service must contain an extensible driver-based
architecture, so that it is capable of sending notifications to a
number of different recovery workflow controllers.

In particular it must have a driver for sending notifications via the
`masakari API <https://github.com/openstack/masakari>`_.  If the
service is implemented as a shell script, this could be achieved by
invoking masakari's ``notification-create`` CLI, or if in Python, via
the `python-masakariclient library
<https://github.com/openstack/python-masakariclient>`_.

Ideally it should also have a driver for sending HTTP POST messages to
a configurable endpoint with JSON data formatted in the following
form:

.. code-block:: json

    {
        "id": UUID,
        "event_type": "host failure",
        "version": "1.0",
        "generated_time" : TIMESTAMP,
        "payload": {
            "hostname": COMPUTE_NAME
            "on_shared_storage": [true|false],
            "failure_time" : TIMESTAMP
        },
    }

``COMPUTE_NAME`` refers to the FQDN of the compute node on which the
failures have occurred.  ``on_shared_storage`` is ``true`` if and only
if the compute host's instances are backed by shared storage.
``failure_time`` provides a timestamp (in seconds since the UNIX
epoch) for when the failure occurred.

This is already implemented as `fence_evacuate.py
<https://github.com/gryf/mistral-evacuate/blob/master/fence_evacuate.py>`_,
although the message sent by that script is currently specifically
formatted to be consumed by Mistral.

Alternatives
============

No alternatives to the overall architecture are obviously apparent at
this point.  However it is possible that the use of `attrd
<http://clusterlabs.org/man/pacemaker/attrd_updater.8.html>`_ (which
is functional but not comprehensively documented) could be substituted
for some other highly available key/value attribute store, such as
`etcd <https://coreos.com/etcd>`_.

Impact assessment
=================

Data model impact
-----------------

None

API impact
----------

The HTTP API of the host recovery workflow service needs to be able to
receive events in the format they are sent by this host monitor.

Security impact
---------------

Ideally it should be possible for the host monitor to send
instance event data securely to the recovery workflow service
(e.g. via TLS), without relying on the security of the admin network
over which the data is sent.

Other end user impact
---------------------

None

Performance Impact
------------------

There will be a small amount of extra RAM and CPU required on each
compute node for running the ``pacemaker_remote`` service.  However
it's a relatively simple service, so this should not have significant
impact on the node.

Other deployer impact
---------------------

Distributions need to package ``pacemaker_remote``; however this is
already done for many distributions including SLES, openSUSE, RHEL,
CentOS, Fedora, Ubuntu, and Debian.

Automated deployment solutions need to deploy and configure the
``pacemaker_remote`` service on each compute node; however this is a
relatively simple task.

Developer impact
----------------

Nothing other than the listed work items below.

Documentation Impact
--------------------

The service should be documented in the |ha-guide|_.

Assignee(s)
===========

Primary assignee:

- Adam Spiers

Other contributors:

- Sampath Priyankara
- Andrew Beekhof
- Dawid Deja

Work Items
==========

- Implement ``nova-host-alerter`` (**TODO**: choose owner for this)

- If appropriate, move the existing `fence_evacuate.py
  <https://github.com/gryf/mistral-evacuate/blob/master/fence_evacuate.py>`_
  to a more suitable long-term home (**TODO**: choose owner for this)

- Add SSL support (**TODO**: choose owner for this)

- Add documentation to the |ha-guide|_ (``aspiers`` / ``beekhof``)

.. |ha-guide| replace:: OpenStack High Availability Guide
.. _ha-guide: http://docs.openstack.org/ha-guide/

Dependencies
============

- `Pacemaker <http://clusterlabs.org/>`_

Testing
=======

`Cloud99 <https://github.com/cisco-oss-eng/Cloud99>`_ could
possibly be used for testing.

References
==========

- `Architecture diagram presented at OpenStack Day Israel, June 2017
  <https://aspiers.github.io/openstack-day-israel-2017-compute-ha/#/nova-host-alerter>`_
  (see also `the video of the talk <https://youtu.be/uMCMDF9VkYk?t=20m9s>`_)

- `"High Availability for Virtual Machines" user story
  <http://specs.openstack.org/openstack/openstack-user-stories/user-stories/proposed/ha_vm.html>`_

- `Video of "High Availability for Instances: Moving to a Converged Upstream Solution"
  presentation at OpenStack conference in Boston, May 2017
  <https://www.openstack.org/videos/boston-2017/high-availability-for-instances-moving-to-a-converged-upstream-solution>`_

- `Instance HA etherpad started at Newton Design Summit in Austin, April 2016
  <https://etherpad.openstack.org/p/newton-instance-ha>`_

- `Video of "HA for Pets and Hypervisors" presentation at OpenStack conference
  in Austin, April 2016
  <https://www.openstack.org/videos/video/high-availability-for-pets-and-hypervisors-state-of-the-nation>`_

- `automatic-evacuation etherpad
  <https://etherpad.openstack.org/p/automatic-evacuation>`_

- Existing `fence agent
  <https://github.com/gryf/mistral-evacuate/blob/master/fence_evacuate.py>`_
  which sends failure notification payload as JSON over HTTP.

- `Instance auto-evacuation cross project spec (WIP)
  <https://review.openstack.org/#/c/257809>`_


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Pike
     - Updated to have alerting mechanism decoupled from fencing process
   * - Newton
     - First introduced
