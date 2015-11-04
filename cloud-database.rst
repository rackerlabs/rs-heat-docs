==========================
 Rackspace Cloud Databases
==========================

Brief summary
=============

Rackspace Cloud Databases can be created, updated, and deleted using
the OS::Trove::Instance resource.  Cloud Databases instances can also
be created as replicas of other Cloud Databases instances.

Example template
================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Create a Rackspace Cloud Database instance and make a replica.

    resources:

    outputs:

Resources section
-----------------

Add an OS::Trove::Instance resource with a list of databases and
users:

.. code:: yaml

      db:
        type: OS::Trove::Instance
        properties:
          name: db
          flavor: 1GB Instance
          size: 10
          databases:
          - name: my_data
          users:
          - name: john
            password: secrete
            databases: [ my_data ]

This resource will create your Cloud Databases instance.

Add another OS::Trove::Instance, but this time leave out the databases
and users and specify a ``replica_of`` property:

.. code:: yaml

      db_replica:
        type: OS::Trove::Instance
        properties:
          name: db_replica
          flavor: 1GB Instance
          size: 10
          replica_of: { get_resource: db }

This will create a replica of your first Cloud Databases instance.
Alternatively, you can add a template parameter for the UUID of the
database instance that you want a replica of and pass in the UUID upon
stack creation.

Outputs section
---------------

Add the following to your outputs section:

.. code:: yaml

      "DB ID":
        value: { get_resource: db }
        description: Database instance ID.
      
      "DB hostname":
        value: { get_attr: [db, hostname] }
        description: Database instance hostname.
      
      "DB href":
        value: { get_attr: [db, href] }
        description: Api endpoint of the database instance.
      
      "DB replica ID":
        value: { get_resource: db_replica }
        description: Database replica ID.
      
      "DB replica hostname":
        value: { get_attr: [db_replica, hostname] }
        description: Database replica hostname.
      
      "DB replica href":
        value: { get_attr: [db_replica, href] }
          description: Api endpoint of the database replica.
    
Full template
-------------

.. code:: yaml

    heat_template_version: 2014-10-16
    
    description: |
      Test template using Trove with replication
    
    resources:
      db:
        type: OS::Trove::Instance
        properties:
          name: db
          flavor: 1GB Instance
          size: 10
          databases:
          - name: my_data
          users:
          - name: john
            password: secrete
            databases: [ my_data ]
    
      db_replica:
        type: OS::Trove::Instance
        properties:
          name: db_replica
          flavor: 1GB Instance
          size: 10
          replica_of: { get_resource: db }

    outputs: 
      "DB ID":
        value: { get_resource: db }
        description: Database instance ID.

      "DB hostname":
        value: { get_attr: [db, hostname] }
        description: Database instance hostname.

      "DB href":
        value: { get_attr: [db, href] }
        description: Api endpoint of the database instance.

      "DB replica ID":
        value: { get_resource: db_replica }
        description: Database replica ID.

      "DB replica hostname":
        value: { get_attr: [db_replica, hostname] }
        description: Database replica hostname.

      "DB replica href":
        value: { get_attr: [db_replica, href] }
        description: Api endpoint of the database replica.


Reference documentation
=======================

- `OS::Trove::Instance <http://docs.openstack.org/hot-reference/content/OS__Trove__Instance.html>`__
- `Rackspace Cloud Databases Developer Guide <https://developer.rackspace.com/docs/cloud-databases/v1/developer-guide/>`__
