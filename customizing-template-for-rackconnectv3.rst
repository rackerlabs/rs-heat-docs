======================================================================
Customizing Rackspace supported templates for Rackconnect V3 customers
======================================================================

Note: This document assumes that the reader is familiar with HOT
specification. If that is not the case, please go to 'References'
section given at the end of this document for HOT specification link.

Brief summary
=============

`Rackspace supported templates <https://github.com/rackspace-orchestration-templates>`__
are not currently supported for Rackconnect V3 customers. This document outlines the steps needed to
make a template work in Rackconnected V3 account.

Prerequisite
============
Some of the rackspace supported templates use ChefSolo resource. If you are customizing
a template that cotains ChefSolo resource, make sure that rack connected servers can access the internet.
This is required because ChefSolo resource downloads chef from internet. Please contact the Rackconnect customer service
to update outbound NAT for your rackconnect account.

Customizing a template
======================

1. Clone the template repository you wanted to customize into your public personal github account. This
repository must be accessible to public without any authentication

2. Template repository may have multiple template files(Template can have multiple child templates). Find
all the template files ending with .yaml (except rackspace.yaml)

3. In the template files, find all the places where 'OS::Nova::Server' resource is being used and provide servicenet
and rackconnect networks to that server resource

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

4. Find all the references to 'OS::Heat::ChefSolo' resource and use servicenet/private IP of the rackconnected
server instead of public IP

5. Inside the template, if any of the rackconnected server is connecting/communicating with other rackconnected
server, then use rackconnected IP instead of servicenet or public IP.

Example (customized template)
=============================
For example consider customizing mongodb template.

1. Rackspace supported mongodb template which doesn't work for Rackconnect V3 customers is available
at `<https://github.com/rackspace-orchestration-templates/mongodb-replset>`__

2. Cloned and customized template repository is available 
at `<https://github.com/vikomall/mongodb-replset>`__

3. List of changes made to the original template can be seen at https://github.com/rackspace-orchestration-templates/mongodb-replset/compare/master...vikomall:master


Reference
=========

-  `Cloud Orchestration API Developer
   Guide <http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/overview.html>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Rackconnect compatibility
   information <http://www.rackspace.com/knowledge_center/article/rackconnect-v30-compatibility>`__
-  `Orchestration support for rackconnect v3 <http://www.rackspace.com/knowledge_center/article/cloud-orchestration-support-for-rackconnect-v30>`__
