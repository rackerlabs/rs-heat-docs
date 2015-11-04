======================================================================
Customizing Rackspace supported templates for RackConnect V3 customers
======================================================================

Note: This document assumes that the reader is familiar with HOT
specification. If that is not the case, please go to the References
section given at the end of this tutorial for the HOT specification link.

Brief summary
=============

`Rackspace supported templates <https://github.com/rackspace-orchestration-templates>`__
are not currently supported for RackConnect V3 customers. This document outlines the steps needed to
make a template work in a rackconnected V3 account.

Prerequisite
============
Some of the Rackspace supported templates use the ChefSolo resource. If you are customizing
a template that contains the ChefSolo resource, make sure that rackconnected servers can access the internet.
This is required because the ChefSolo resource downloads chef from the internet. Please contact the 
RackConnect Customer Service to update outbound NAT for your RackConnect account.

Customizing a template
======================

1. Clone the template repository you want to customize into your public personal github account. This
repository must be accessible to the public without any authentication.

2. A template repository may have multiple template files (a template can have multiple child templates). Find
all the template files ending with .yaml (except ``rackspace.yaml``).

3. In the template files, find all the places where the OS::Nova::Server resource is being used and provide servicenet
and RackConnect networks to that server resource.

.. code:: yaml

      server:
        type: "OS::Nova::Server"
        properties:
          name: test-server
          flavor: 2 GB General Purpose v1
          image: Debian 7 (Wheezy) (PVHVM)
          networks:
          - network: <rackconnect_network_name>
          - uuid: 11111111-1111-1111-1111-111111111111

4. Find all the references to the OS::Heat::ChefSolo resource and use the servicenet/private IP of the rackconnected
server instead of the public IP.

5. Inside the template, if any rackconnected server is connecting/communicating with other rackconnected
servers, then use the rackconnected IP instead of the servicenet or public IP.

Example (customized template)
=============================
For example consider customizing the mongodb template.

1. Rackspace supported mongodb template that doesn't work for RackConnect V3 customers is available
at `<https://github.com/rackspace-orchestration-templates/mongodb-replset>`__

2. Cloned and customized template repository is available 
at `<https://github.com/vikomall/mongodb-replset>`__

3. List of changes made to the original template can be seen at https://github.com/rackspace-orchestration-templates/mongodb-replset/compare/master...vikomall:master


Reference
=========

-  `Cloud Orchestration API Developer
   Guide <https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide/>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `RackConnect compatibility
   information <http://www.rackspace.com/knowledge_center/article/rackconnect-v30-compatibility>`__
-  `Orchestration support for RackConnect v3 <http://www.rackspace.com/knowledge_center/article/cloud-orchestration-support-for-rackconnect-v30>`__
