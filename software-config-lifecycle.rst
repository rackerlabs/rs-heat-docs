.. _software_config_lifecycle:

=========================
Software Config Lifecycle
=========================

Brief summary
=============

One of the more interesting and helpful features of software config is the ability to
run deployments only during certain lifecycle events. This allows the template author
to run scripts and other configurations only during specific actions like `DELETE` or
`UPDATE`. In this example, we will use the the ``actions`` property of
``OS::Heat::SoftwareDeployment`` to apply specific configurations only during the listed
stack lifecycle actions.

One of the trickier things about using servers with attached Cloud Block Storage volumes
is that in order to detatch the volume from the server, the volume must be unmounted.
Usually, this means having to manually unmount the volume before detatching and deleting
the other resources. Using software config during specific stack actions allows us to
easily automate this process.

You can find the full template for this example at `this repository's
templates directory <https://github.com/rackerlabs/rs-heat-docs/blob/master/templates/volume_management.yaml>`_.

Pre-reading
===========

The following introductory material should give you enough background to
proceed with this tutorial.

-  `Application software configuration using
   Heat <https://www.openstack.org/assets/presentation-media/heat-software-config.pdf>`__
-  `HOT guide - Software
   configuration <http://docs.openstack.org/developer/heat/template_guide/software_deployment.html>`__
-  `Software Config example
   templates <https://github.com/openstack/heat-templates/tree/master/hot/software-config/example-templates>`__

Example template
================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16 

    description: |
      Mounting/unmounting volumes with software config

    resources:

    outputs:

Resources section
-----------------

Add the server that we'll manage Cloud Block Storage volumes on:

.. code:: yaml

        # server to attach a volume to
        server:
          type: OS::Nova::Server
          properties:
            name:
              str_replace:
                template: stack-volume-server
                params:
                  stack: { get_param: "OS::stack_name" } 
            metadata:
              rax-heat: { get_param: "OS::stack_id" }
            # use a public image that already has all the agents set up for software
            # deployments; could also use a custom image or bootstrap a "clean" one
            image: f4bbbce2-50b0-4b07-bf09-96c175a45f4b # Ubuntu 14.04 with agents
            flavor: 2 GB Performance
            config_drive: true
            software_config_transport: POLL_TEMP_URL
            user_data_format: SOFTWARE_CONFIG

Here, we've used a public image that already has all of the required software config
agents installed, but you can use a custom image (such as the one you may have created
in the `Bootstrapping Software Config` tutorial), or bootstrap a "clean" image from
the catalog. Also note that we've set up ``config_drive``, ``software_config_transport``
and ``user_data_format`` as required.

Now, we'll add a volume and attach it to the server:

.. code:: yaml

        # volume to attach to the server
        volume:
          type: OS::Cinder::Volume
          properties:
            name:
              str_replace:
                template: stack-test-volume
                params:
                  stack: { get_param: "OS::stack_name" } 
            metadata:
              rax-heat: { get_param: "OS::stack_id" }
            size: 100
            description: Volume for testing management via software config

        # attach the volume to the server
        attach_vol:
          type: OS::Cinder::VolumeAttachment
          properties:
            instance_uuid: { get_resource: server }
            volume_id: { get_resource: volume }
            mountpoint: "/dev/xvdb"

Notice the ``mountpoint`` property; we'll use this in a bit in our configuration.

Next, we'll add the software configurations to run on the server to manage the volume:

.. code:: yaml

      # script to configure and mount the volume
      config_volume:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          config: |
            #!/bin/bash
            (echo n; echo p; echo 1; echo; echo; echo w;) | fdisk /dev/xvdb
            mkfs -t ext4 /dev/xvdb1
            mkdir -p /myvol
            mount /dev/xvdb1 /myvol
            echo "/dev/xvdb1 /myvol ext4 defaults,noatime,_netdev,nofail 0 2" >> /etc/fstab

      # script to unmount the volume
      unmount_vol:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          config: |
            #!/bin/bash
            umount -l /myvol

The ``config_volume`` configuration is designed to run after the server is created to
format and mount the volume. Notice that it uses the ``mountpoint`` we defined in the 
``attach_vol`` resource previously.

The ``unmount_vol`` is designed to run before the volume is detached and deleted. This is
important because if we did not unmount the volume prior to detaching it, the stack would
fail to delete.

Now, lets add a deployment resources that will execute these configurations at the
appropriate times in the stack's lifecycle. First, we'll deploy ``config_volume`` to 
the server during the ``CREATE`` phase:

.. code:: yaml

      # run the script to setup and mount the volume once the server is up and the volume
      # is attached
      prep_volume:
        type: OS::Heat::SoftwareDeployment
        depends_on: attach_vol # make sure this runs after attach
        properties:
          signal_transport: TEMP_URL_SIGNAL
          config: { get_resource: config_volume }
          server: { get_resource: server }
          actions:
          - CREATE # only run on stack-create

Notice that we explicitly depend on the ``attach_vol`` resource so that we are sure
the volume is attached and available before we try to configure it. Also notice that we
use the ``actions`` property to tell the orchestration engine to only consider this
resource during the ``CREATE`` phase. This deployment will therefor be ignored during
stack ``UPDATE``, ``DELETE`` or any other lifecycle operation that isn't ``CREATE``.

Lastly, we'll deploy the ``unmount_vol`` configuration to the server during stack
``DELETE``:

.. code:: yaml

      # before detaching the volume, run the script that unmounts it
      pre_delete:
        type: OS::Heat::SoftwareDeployment
        depends_on: attach_vol # make sure this runs before detach
        properties:
          signal_transport: TEMP_URL_SIGNAL
          config: { get_resource: unmount_vol }
          server: { get_resource: server }
          actions:
          - DELETE # only run on stack-delete

Note that this resource also depends on the ``attach_vol`` resource. This is because we
want to execute the configuration *before* the volume is detached. This works because
during ``DELETE`` operations, Orchestration works through the dependency tree in reverse.
This allows us to unmount our volume prior to detaching it and deleting the other
resources.

Full template
-------------

You can find the full template for this example at `this repository's
templates directory <https://github.com/rackerlabs/rs-heat-docs/blob/master/templates/volume_management.yaml>`_.


Reference documentation
=======================

- `OS::Heat::SoftwareConfig <http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareConfig>`__
- `OS::Heat::SoftwareDeployment <http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareDeployment>`__
