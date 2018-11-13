.. _newton-priorities:

=========================
Newton Project Priorities
=========================

List of themes (in the form of use cases) the OpenStack HA community
is prioritizing in Newton.

+-------------------------------------------+-----------------------+
| Priority                                  | Primary Contacts      |
+===========================================+=======================+
| `Highly Available VMs`_                   | `Adam Spiers`_        |
+-------------------------------------------+-----------------------+

Highly Available VMs
--------------------

The OpenStack `Product Working Group`_ is maintaining a user story for
`High Availability for Virtual Machines`_.  During and since the
Austin summit, there has been a lot of discussion within the OpenStack
HA community about how to converge on an upstream implementation which
fulfils this user story, which has been captured in the
`newton-instance-ha etherpad`_ and in the minutes of the `weekly HA
IRC meetings`_.  The first step is to write a series of specs which
will describe the architectural design and corresponding deliverables
for each of the components which were agreed as necessary for a
flexible, modular solution.

.. _Adam Spiers: https://launchpad.net/~adam.spiers
.. _High Availability for Virtual Machines: https://specs.openstack.org/openstack/openstack-user-stories/user-stories/proposed/ha_vm.html
.. _Product Working Group: https://wiki.openstack.org/wiki/ProductTeam
.. _weekly HA IRC meetings: https://wiki.openstack.org/wiki/Meetings/HATeamMeeting
.. _newton-instance-ha etherpad: https://etherpad.openstack.org/p/newton-instance-ha
