.. _generic-software-config:

=========================
 Generic software config
=========================

Brief summary
=============

If you have ever needed to configure a server with Heat, chances are you
have written a user_data script inside of a OS::Nova::Server
resource. The OS::Heat::SoftwareConfig resource is another way to
configure a server. It provides several advantages over defining a
user_data script.

One SoftwareConfig resource can be associated with many servers. Each
time it is triggered, it can be run with different parameters. In
addition, a SoftwareConfig resource can be updated and rerun without
causing the server to be rebuilt.

SoftwareConfig can also be used to configure your server using 
configuration management software, such as Ansible or Puppet. In this
tutorial, we will configure the server with a simple shell script.

Pre-reading
===========

The following introductory material should give you enough background to
proceed with this tutorial.

-  `Heat SoftwareConfig resources -
   primer/overview <http://hardysteven.blogspot.com/2015/05/heat-softwareconfig-resources.html>`__
   by `Steven Hardy <http://hardysteven.blogspot.com/>`__ (licensed under `CC SA 3.0 <http://creativecommons.org/licenses/by-sa/3.0/legalcode>`__)
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
      A template which demonstrates doing boot-time installation of the required
      files for script based software deployments.
      This template expects to be created with an environment which defines
      the resource type Heat::InstallConfigAgent such as
      ../boot-config/fedora_pip_env.yaml

    parameters:

    resources:

    outputs:

Parameters section
------------------

Add a template parameter for the server image:

.. code:: yaml

      image:
        type: string

Resources section
-----------------

Add an OS::Heat::SoftwareConfig resource, which will be used to define a
software configuration.

.. code:: yaml

      config:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          inputs:
          - name: foo
          - name: bar
          outputs:
          - name: result
          config: |
            #!/bin/sh -x
            echo "Writing to /tmp/$bar"
            echo $foo > /tmp/$bar
            echo -n "The file /tmp/$bar contains `cat /tmp/$bar` for server $deploy_server_id during $deploy_action" > $heat_outputs_path.result
            echo "Written to /tmp/$bar"
            echo "Output to stderr" 1>&2

The "group" property is used to specify the type of SoftwareConfig
hook that will be used to deploy the configuration. Other
SoftwareConfig hooks are available in the `openstack/heat-templates
repository
<https://github.com/openstack/heat-templates/tree/master/hot/software-config/elements>`__
on GitHub.

Add an OS::Heat::SoftwareDeployment resource, which will be used to
associate a SoftwareConfig resource and a set of input values with the
server to which it will be deployed.

.. code:: yaml

      deployment:
        type: OS::Heat::SoftwareDeployment
        properties:
          signal_transport: TEMP_URL_SIGNAL
          config:
            get_resource: config
          server:
            get_resource: server
          input_values:
            foo: fooooo
            bar: baaaaa

It is advisable to specify a "signal_transport" of "TEMP_URLSIGNAL",
because Rackspace's deployment of Heat does not support the other
transports at this time. However, since this is the default
transport on the Rackspace Cloud, it should be safe to omit.

Add an InstallConfigAgent resource, which will be mapped via the
environment to a `"provider" resource
<http://hardysteven.blogspot.com/2013/10/heat-providersenvironments-101-ive.html>`__:

.. code:: yaml

      boot_config:
        type: Heat::InstallConfigAgent

The purpose of this resource is to provide output for the user_data
section that will be used to install the config agent on the Server
resource below. See the :ref:`Usage <generic-sw-config-usage>` section below for more
information on using this resource.

Add a Nova server key pair resource as a way to access the server to
confirm deployment results:

.. code:: yaml

      ssh_key:
        type: OS::Nova::KeyPair
        properties:
          name: private_access_key
          save_private_key: true

Finally, add the OS::Nova::Server resource and reference the
boot_config resource in the user_data section:

.. code:: yaml

      server:
        type: OS::Nova::Server
        properties:
          image: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf  # Ubuntu 14.04
          flavor: 2 GB Performance
          key_name: { get_resource: ssh_key }
          software_config_transport: POLL_TEMP_URL
          user_data_format: SOFTWARE_CONFIG
          user_data: {get_attr: [boot_config, config]}

Outputs section
---------------

Add the following to your outputs section:

.. code:: yaml

      result:
        value:
          get_attr: [deployment, result]
      stdout:
        value:
          get_attr: [deployment, deploy_stdout]
      stderr:
        value:
          get_attr: [deployment, deploy_stderr]
      status_code:
        value:
          get_attr: [deployment, deploy_status_code]
      server_ip:
        value:
          get_attr: [server, accessIPv4]
      private_key:
        value:
          get_attr: [ssh_key, private_key]

This will show the actual script output from the SoftwareConfig
resource.

Full template
-------------

.. code:: yaml

    heat_template_version: 2014-10-16
    description: |
      A template which demonstrates doing boot-time installation of the required
      files for script based software deployments.
      This template expects to be created with an environment which defines
      the resource type Heat::InstallConfigAgent such as
      ../boot-config/fedora_pip_env.yaml

    parameters:

      image:
        type: string

    resources:

      config:
        type: OS::Heat::SoftwareConfig
        properties:
          group: script
          inputs:
          - name: foo
          - name: bar
          outputs:
          - name: result
          config: |
            #!/bin/sh -x
            echo "Writing to /tmp/$bar"
            echo $foo > /tmp/$bar
            echo -n "The file /tmp/$bar contains `cat /tmp/$bar` for server $deploy_server_id during $deploy_action" > $heat_outputs_path.result
            echo "Written to /tmp/$bar"
            echo "Output to stderr" 1>&2

      deployment:
        type: OS::Heat::SoftwareDeployment
        properties:
          signal_transport: TEMP_URL_SIGNAL
          config:
            get_resource: config
          server:
            get_resource: server
          input_values:
            foo: fooooo
            bar: baaaaa

      boot_config:
        type: Heat::InstallConfigAgent

      ssh_key:
        type: OS::Nova::KeyPair
        properties:
          name: private_access_key
          save_private_key: true

      server:
        type: OS::Nova::Server
        properties:
          image: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf  # Ubuntu Ubuntu 14.04
          flavor: 2 GB Performance
          key_name: { get_resource: ssh_key }
          software_config_transport: POLL_TEMP_URL
          user_data_format: SOFTWARE_CONFIG
          user_data: {get_attr: [boot_config, config]}

    outputs:
      result:
        value:
          get_attr: [deployment, result]
      stdout:
        value:
          get_attr: [deployment, deploy_stdout]
      stderr:
        value:
          get_attr: [deployment, deploy_stderr]
      status_code:
        value:
          get_attr: [deployment, deploy_status_code]
      server_ip:
        value:
          get_attr: [server, accessIPv4]
      private_key:
        value:
          get_attr: [ssh_key, private_key]

.. _generic-sw-config-usage:

Usage
=====

Before you create the stack, you need an environment file that will define
a Heat::InstallConfigAgent resource to tell Heat how to install the
config agent on Ubuntu 14.04.

First, clone the heat-templates repository:

.. code::

    git clone https://github.com/openstack/heat-templates.git

The environment file you will use is located under
``heat-templates/hot/software-config/boot-config/ubuntu_pip_env.yaml``.
It will supply the image parameter to the template. A ready-made
InstallConfigAgent resource for Fedora also exists in the heat-templates
repository in case you want to use Fedora.

Then, issue the ``stack-create`` command with the template and environment
file just created using python-heatclient:

.. code::

    heat --heat-url=https://dfw.orchestration.api.rackspacecloud.com/v1/$RS_ACCOUNT_NUMBER --os-username $RS_USER_NAME --os-password $RS_PASSWORD --os-tenant-id $RS_ACCOUNT_NUMBER --os-auth-url https://identity.api.rackspacecloud.com/v2.0/ stack-create -f generic-software-config.yaml -e heat-templates/hot/software-config/boot-config/ubuntu_pip_env.yaml generic-software-config1

Next, edit the template and perform a ``stack-update``. Edit the
SoftwareDeployment parameters in the template:

.. code::

    sed -i.bak -e 's/fooooo/fooooo1/' -e 's/baaaaa/baaaaa1/' generic-software-config.yaml 

Issue the ``stack-update`` command:

.. code::

    heat --heat-url=https://dfw.orchestration.api.rackspacecloud.com/v1/$RS_ACCOUNT_NUMBER --os-username $RS_USER_NAME --os-password $RS_PASSWORD --os-tenant-id $RS_ACCOUNT_NUMBER --os-auth-url https://identity.api.rackspacecloud.com/v2.0/ stack-update -f generic-software-config.yaml -e heat-templates/hot/software-config/boot-config/ubuntu_pip_env.yaml generic-software-config1

Notice that the config agent re-runs the script without rebuilding the
server. In a couple of minutes, a new file should exist alongside the
original one: ``/tmp/fooooo1`` with the content ``baaaaa1``.

Reference documentation
=======================

- `OS::Heat::SoftwareConfig <http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareConfig>`__
- `OS::Heat::SoftwareDeployment <http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareDeployment>`__
