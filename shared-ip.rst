===================================
 Rackspace shared IP resource
===================================

Brief summary
=============

Rackspace shared IP resource (SharedIP) and AssociateSharedIP resources
enable you to create a shared IP address and assoicate the shared IP
address with two or more virtual server instances.

Steps involved
---------------
The following steps are involved in using the shared IP address.

#. Create (two or more) servers in the same publicIPzoneId and write
   down the public IP address ports.
   For more information see
   http://docs-beta.rackspace.com/drafts/catlook/webhelp/cn-gettingstarted-external/content/share_ip.html.

#. Create a shared IP address with the given network ID and port IDs.

#. Associate shared IPs with the servers.

In the following example template we create a shared IP address and associate
it with two server instances. For the sake of simplicity, we will
assume that two servers were already created in the same publicIPzoneId.

Example template
=================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Shared IP example template.

    resources:

    outputs:

Resources section
=================

SharedIP resource
-----------------

Add a Rackspace::Cloud::SharedIP resource to create a shared IP address.

.. code:: yaml

    shared_ip:
        properties:
            network_id: 00000000-0000-0000-0000-000000000000
            ports: [55xxfxx6-cxx7-4xxb-8xx3-3cxxd12xxe0d, 17xxfxxca-exx2-4xxe-bxx7-91xxf6xxbb2]
        type: Rackspace::Cloud::SharedIP


Here the public network ID 00000000-0000-0000-0000-000000000000 is
provided as the ``network_id`` property value, and public IP port IDs
55xxfxx6-cxx7-4xxb-8xx3-3cxxd12xxe0d and 17xxfxxca-exx2-4xxe-bxx7-91xxf6xxbb2 are
provided as a list to the ``ports`` property.

Please refer to the 'Steps Involved' section listed above for creating
servers and getting the port IDs.

AssociateSharedIP resource
----------------------------

Add a Rackspace::Cloud::AssociateSharedIP resource to associate a
shared IP address with the given server instances.

.. code:: yaml

    associate_shared_ip:
        properties:
            shared_ip: {get_attr: [shared_ip, shared_ip_address, ip_address, id]}
            servers: [62cxx03b-axx7-4xxb-bxxb-f1axx14370b4, 6exx610f-1xx2-4xx9-9xx5c-bxx2c735e463]
        type: Rackspace::Cloud::AssociateSharedIP

Here 62cxx03b-axx7-4xxb-bxxb-f1axx14370b4 and 6exx610f-1xx2-4xx9-9xx5c-bxx2c735e463
are the server instance IDs (please note these are not port IDs) and they are passed as a
list to the ``servers`` property.

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

-  `Cloud Orchestration API Developer
   Guide <https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide/>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Share IP address between
   servers <http://docs-beta.rackspace.com/drafts/catlook/webhelp/cn-gettingstarted-external/content/share_ip.html>`__
-  `IP addresses
   extension <http://docs-beta.rackspace.com/drafts/catlook/webhelp/cn-devguide-external/content/api_ext_sharedip_neutron.html>`__

.. comment: Writer: need to update links for previous 2 bullets in late Nov. 2015 to point to correct sections in public Network Guide.