===================================
 Rackspace shared IP resource
===================================

Brief summary
=============

You can use the Rackspace shared IP resource (SharedIP) and AssociateSharedIP resources
to create a shared IP address and associate the shared IP address with two or more
virtual server instances.

.. _setup-process:

Setup process
-------------

The following steps describe the process to set up and use a shared IP
address between servers.

.. note:: 

   For additional information, see `Share IP address between servers`_ in the Rackspace 
   Cloud Networks documentation.

#. Create (two or more) servers in the same ``publicIPzoneId`` and write
   down the public IP address ports.
   
#. Create a shared IP address with the given network ID and port IDs.

#. Associate shared IPs with the servers.

The following example template provides the code to create a shared IP address and associate
it with two server instances. For the sake of simplicity, assume that 
two servers were already created in the same ``publicIPzoneId``.

Example template
=================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Shared IP example template.

    resources:

    outputs:

Resources
=========

The following sections provide information about the resources, outputs, and an example of the full template to set up
a shared IP address between servers.

SharedIP resource
-----------------

Add a Rackspace::Cloud::SharedIP resource to create a shared IP address.

.. code:: yaml

    shared_ip:
        properties:
            network_id: 00000000-0000-0000-0000-000000000000
            ports: [55xxfxx6-cxx7-4xxb-8xx3-3cxxd12xxe0d, 17xxfxxca-exx2-4xxe-bxx7-91xxf6xxbb2]
        type: Rackspace::Cloud::SharedIP


The ``network_id`` property provides the value for the public network ID,``00000000-0000-0000-0000-000000000000``.
The ``ports`` property specifies a list of public port IDs, ``55xxfxx6-cxx7-4xxb-8xx3-3cxxd12xxe0d`` and ``17xxfxxca-exx2-4xxe-bxx7-91xxf6xxbb2``.


For information about creating a server and getting port IDs, see the :ref:`Setup process <setup-process>`.

AssociateSharedIP resource
----------------------------

Add a ``Rackspace::Cloud::AssociateSharedIP`` resource to associate a
shared IP address with the given server instances.

.. code:: yaml

    associate_shared_ip:
        properties:
            shared_ip: {get_attr: [shared_ip, shared_ip_address, ip_address, id]}
            servers: [62cxx03b-axx7-4xxb-bxxb-f1axx14370b4, 6exx610f-1xx2-4xx9-9xx5c-bxx2c735e463]
        type: Rackspace::Cloud::AssociateSharedIP

The ``servers`` property specifies a list of the server instance IDs:
``62cxx03b-axx7-4xxb-bxxb-f1axx14370b4`` and ``6exx610f-1xx2-4xx9-9xx5c-bxx2c735e463``.
Note that these values are not port IDs.

Outputs section
---------------

Add the shared IP address to the outputs section.

.. code:: yaml

    shared_ip_address:
        value:
            get_attr: [shared_ip, shared_ip_address]



Full Example Template
---------------------

.. code:: yaml

    heat_template_version: 2014-10-16
    
    description: |
      Shared IP example template.
    
    outputs:
        shared_ip_address:
            value:
                get_attr: [shared_ip, shared_ip_address]
    resources:
        server1:
            type: OS::Nova::Server
            properties:
                image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                flavor: 2 GB Performance
    
        server2:
            type: OS::Nova::Server
            properties:
                image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                flavor: 2 GB Performance
    
        shared_ip:
            properties:
                network_id: 00000000-0000-0000-0000-000000000000
                ports: [{ get_attr: [ server1, addresses, public, 0, port ] }, { get_attr: [ server2, addresses, public, 0, port ] }]
            type: Rackspace::Cloud::SharedIP
    
        associate_shared_ip:
            properties:
                shared_ip: {get_attr: [shared_ip, shared_ip_address, ip_address, id]}
                servers: [{get_resource: server1}, {get_resource: server2}]
            type: Rackspace::Cloud::AssociateSharedIP

Reference
=========

-  `Cloud Orchestration API Developer Guide`_
-  `Heat Orchestration Template (HOT)`_
-  `Share IP address between servers`_
-  `Shared IP address operations`_
   
   
   .. comment Following are reference definitions for links in above text.
   
   .. _Cloud Orchestration API Developer Guide:
      https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide
      
   .. _Shared IP address operations:
      https://developer.rackspace.com/docs/cloud-networks/v2/developer-guide/#shared-ip-address-operations
      
   .. _Share IP address between servers:
      https://developer.rackspace.com/docs/cloud-networks/v2/developer-guide/#sharing-ip-address-between-servers
      
   .. _Heat Orchestration Template (HOT): 
      http://docs.openstack.org/developer/heat/template_guide/hot_spec.html
