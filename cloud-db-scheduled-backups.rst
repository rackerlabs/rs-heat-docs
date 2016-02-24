================================================
 Rackspace Cloud Databases and Scheduled Backups
================================================

Brief summary
=============

Cloud Databases allows you to create a schedule for running a weekly backup for your
database instance. There is an incremental backup run at the end of every day and a full
backup is run on the day as defined by the backup schedule. The backup can always be
restored to a new database instance.

Cloud Orchestration allows you to create, update, and delete these backup schedules by
using the ``Rackspace::CloudDatabase::ScheduledBackup`` resource.

Example template
================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2015-10-15

    description: |
      Simple template to illustrate creating a scheduled backup
      for a Cloud Database instance

    resources:

    outputs:

Resources section
-----------------

We first add an ``OS::Heat::RandomString`` resource to generate a password for the database
user we'll create later:

.. code:: yaml

    # generate a password for our db user
    db_pass:
      type: OS::Heat::RandomString

Next, we add an ``OS::Trove::Instance`` resource with a test database and user. Note we've
set the user's password to the value of the ``OS::Heat::RandomString`` resource we defined
earlier:

.. code:: yaml

    service_db:
      type: OS::Trove::Instance
      properties:
        name: trove_test_db
        datastore_type: mariadb
        datastore_version: 10
        flavor: 1GB Instance
        size: 10
        databases:
        - name: test_data
        users:
        - name: dbuser
          password: { get_attr: [ db_pass, value ] } # use generated password
          databases: [ test_data ]

Lastly, we add the ``Rackspace::CloudDatabase::ScheduledBackup`` resource and configure it
to backup our instance every Monday at 5:45pm and to retain the last 15 full backups:

.. code:: yaml

    backup:
      type: Rackspace::CloudDatabase::ScheduledBackup
      properties:
        source:
          id: { get_resource: service_db }
          type: instance
        day_of_week: 1  # Monday (0-6 Sunday to Saturday)
        hour: 17        # 5pm (24hr clock with 0 being midnight)
        minute: 45      # 5:45pm
        full_backup_retention: 15


Outputs section
---------------

As a convenience, we'll output the user's generated password so we can log in to the 
database if needed:

.. code:: yaml

    "Database Password":
      value: { get_attr: [ db_pass, value ] }
      description: Database password for "dbuser"


Reference documentation
=======================

- You can always view the full template for this guide at `<https://github.com/rackerlabs/rs-heat-docs/tree/master/templates/cloud_db_backups.yaml>`__
- `OS::Trove::Instance <https://developer.rackspace.com/docs/cloud-orchestration/v1/resources-reference/openstack/#os-trove-instance>`__
- `Rackspace::CloudDatabase::ScheduledBackup <https://developer.rackspace.com/docs/cloud-orchestration/v1/resources-reference/openstack/#rackspace-clouddatabase-scheduledbackup>`__
- `Rackspace Cloud Databases Developer Guide <https://developer.rackspace.com/docs/cloud-databases/v1/developer-guide/>`__
