===================================
 SecurityGroup and SecurityGroupAttachment
===================================

Brief summary
=============

SecurityGroupAttachment is used to attach a security group to a port.

SecurityGroupAttachment resource is used in following cases;

(1)If the user wanted to attach a security group to an operator created port

(2)User created a port out side of a template and wanted to attach a
security group to port as part of a template


Limitations / Known Issues
===========================

(1) In Rackspace cloud you cannot apply security groups to a port at boot time.

(2) Security groups can be applied to Rackspace Cloud Servers on Public and ServiceNet Neutron ports. They are not supported for Isolated Networks.

(3) Can only contain rules for the inbound traffic, also known as ingress direction. Outbound traffic, or egress direction, rules are not supported at this time.

(4) Limited to no more than 5 security groups per Neutron port. When a Neutron port has multiple security groups applied, the rules from each security group are effectively aggregated to dictate the rules for access on that port.

(5) The Security Groups API is currently in Limited Availability. It is available only to Managed Infrastructure customers and not to RackConnect or Managed Operations customers. To use this feature, contact Rackspace Support.

Example template
================

In the following example template, we will create linux server and
attach a security group to public network port of the server.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      A linux server with security group attached to public port.

    resources:

    outputs:

Resources section
-----------------

Add a Server resource
~~~~~~~~~~~~~~~~~~~~~

Add a linux server to the template.

.. code:: yaml

      server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance

This creates a server with given image and flavor and also by default attaches public and
service net to the server instance created.

Add SecurityGroup resource
~~~~~~~~~~~~~~~~~~~~~~~~

A security group is a named container for security group rules, which provide
Rackspace Public Cloud users the ability to specify the types of traffic that
are allowed to pass through, to, and from ports (Public/ServiceNet) on
a Cloud server instance.

.. code:: yaml

      security_group:
        type: OS::Neutron::SecurityGroup
        properties:
          name: the_sg
          description: Ping and SSH
          rules:
          - protocol: icmp
          - protocol: tcp
            port_range_min: 22
            port_range_max: 22
          - protocol: tcp
            port_range_min: 5000
            port_range_max: 5000

Here adding a rule for SSH traffic to the security group.

Add SecurityGroupAttachment resource
~~~~~~~~~~~~~~~~~~~~~~~~

Now attach security group to public network port of the server instance.

.. code:: yaml

      security_group_attachment:
        type: OS::Neutron::SecurityGroupAttachment
        properties:
          port: { get_attr: [ server, addresses, public, 0, port ] }
          security_group: {get_resource: security_group}

Here adding a security group to public port of the server instance created.


Full Example Template
---------------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      A linux server with security group attached to public port.

    resources:
      server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance

      security_group:
        type: OS::Neutron::SecurityGroup
        properties:
          name: the_sg
          description: Ping and SSH
          rules:
          - protocol: icmp
          - protocol: tcp
            port_range_min: 22
            port_range_max: 22
          - protocol: tcp
            port_range_min: 5000
            port_range_max: 5000

      security_group_attachment:
        type: OS::Neutron::SecurityGroupAttachment
        properties:
          port: { get_attr: [ server, addresses, public, 0, port ] }
          security_group: {get_resource: security_group}

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/overview.html>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud networks getting started
   documentation <http://docs.rackspace.com/networks/api/v2/cn-gettingstarted/content/ch_preface.html>`__
-  `Cloud networks API 
   documentation <http://docs.rackspace.com/networks/api/v2/cn-devguide/content/ch_preface.html>`__
