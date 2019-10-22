==========================================
 SecurityGroup and SecurityGroupAttachment
==========================================

Brief summary
=============

SecurityGroupAttachment is used to attach a security group to a port.

SecurityGroupAttachment resource is used in the following cases;

(1) If the user wanted to attach a security group to an operator-created port.

(2) The user created a port outside of a template and wanted to attach a
    security group to the port as part of a template.


Limitations / Known Issues
==========================

(1) In Rackspace cloud you cannot apply security groups to a port at boot time.

(2) Security groups can be applied to Rackspace Cloud Servers on Public and ServiceNet Neutron ports. They are not supported for Isolated Networks.

(3) Applying Security Groups to outbound traffic, or egress direction, is supported via the API only (via curl or neutron client).

(4) Limited to no more than 5 security groups per Neutron port. When a Neutron port has multiple security groups applied, the rules from each security group are effectively aggregated to dictate the rules for access on that port.

(5) RackConnect v3 customers are able to use Security Groups if you plan on using Cloud Load Balancers as part of your RackConnected environment. To enable Security Groups on RackConnect v3, please contact Rackspace Support.

Example template
================

In the following example template, we will create a Linux server and
attach a security group to the public network port of the server.

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

Add a Linux server to the template.

.. code:: yaml

      server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance

This creates a server with the given image and flavor and also by default attaches public and
ServiceNet to the server instance created.

Add SecurityGroup resource
~~~~~~~~~~~~~~~~~~~~~~~~~~

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

Here we added a rule for SSH traffic to the security group.

Add SecurityGroupAttachment resource
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now attach security group to the public network port of the server instance.

.. code:: yaml

      security_group_attachment:
        type: Rackspace::Neutron::SecurityGroupAttachment
        properties:
          port: { get_attr: [ server, addresses, public, 0, port ] }
          security_group: {get_resource: security_group}

Here we added a security group to public port of the server instance created.


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
        type: Rackspace::Neutron::SecurityGroupAttachment
        properties:
          port: { get_attr: [ server, addresses, public, 0, port ] }
          security_group: {get_resource: security_group}

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide/>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud networks getting started
   documentation <https://docs.rackspace.com/networks/api/v2/cn-gettingstarted/content/ch_preface.html>`__
-  `Cloud networks API 
   documentation <https://developer.rackspace.com/docs/cloud-networks/v1/developer-guide/>`__
