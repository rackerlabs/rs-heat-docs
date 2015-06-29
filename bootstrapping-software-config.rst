.. role:: bash(code)
   :language: bash

=============================
Bootstrapping Software Config
=============================

Brief summary
=============

In the *Generic Software Config* tutorial, we learned how to use Heat's generic software
configuration mechanism to treat the configuration of compute instances the same way we
treat any other resource in your template. This lesson will go into more detail about
bootstrapping a pristine image for use with software config as well as show how we can
then create our own image with the necessary tools pre-installed for easier use in
future stacks.

Pre-reading
===========

You will want to go over the previous tutorial, *Generic Software Config* first as we'll
be using that example template as a basis for this tutorial. You will also need a very
basic understanding of Heat template composition and Environments.

- `Previous Tutorial <generic_software_config.rst>`_
- `Template Composition Guide
  <http://docs.openstack.org/developer/heat/template_guide/composition.html>_`
- `Environments Guide <http://docs.openstack.org/developer/heat/template_guide/environment.html>`_

Following Along
===============
You will probably want to clone this repository in order to easily follow along.
Otherwise, you may need to modify some of the commands to point to the correct locations
of various templates and environments. You may also have to modify the environment file
to point to the correct `bootconfig_all.yaml`.

Modifying the Example template
==============================

We've started by making a copy of the original example template and saving it as
`software_config_custom_image.yaml`. We'll then removed resources from the resource
section except for the parts that bootstrap the instance as well as the server itself. The
following resources were removed from the template:

- `config`
- `deployment`
- `other_deployment`

We revised the `outputs` section so that we can easily access the server's ip and root
credentials (we'll explain a little more in the next section):

..code:: yaml

outputs:

  server_ip:
    value: { get_attr: [ server, addresses, public, 0, addr ] }
    description: IP address of the server

  admin_password:
    value: { get_attr: [ admin_password, value ] }
    description: Root password to the server

We left the `parameters`, `description`, and `heat_template_version` sections as-is.

Modify the Server
-----------------

We added a `OS::Heat::RandomString` resource to generate a random root password for the
instance so that we can log into the instance after the stack is complete. This is so that
we can make some small modifications later if we want to create an image we can reuse
the next time we want to apply software config to a server.

..code:: yaml

  admin_password:
    type: OS::Heat::RandomString

Since we're not actually deploying any software config to the instance, we can just use
cloud-init to do our installation. To do this, we'll clean up some of this from the server
resource by removing the `software_config_transport` property and changing the
`user_data_format` to `RAW`. We'll also pass in the generated password to the instance:

..code:: yaml

  server:
    type: OS::Nova::Server
    properties:
      image: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf  # Ubuntu Ubuntu 14.04
      flavor: 2 GB Performance
      key_name: { get_resource: ssh_key }
      user_data_format: RAW
      user_data: {get_attr: [boot_config, config]}

Your template should now look like:

..code:: yaml

heat_template_version: 2014-10-16
description: |
  A template that creates a server bootstrapped for use
  with Heat Software Config

parameters:

  image:
    type: string

resources:

  boot_config:
    type: Heat::InstallConfigAgent

  admin_password:
    type: OS::Heat::RandomString

  server:
    type: OS::Nova::Server
    properties:
      name: { get_param: "OS::stack_name" }
      admin_pass: { get_attr: [ admin_password, value ] }
      image: { get_param: image }
      flavor: 2 GB Performance
      user_data_format: RAW
      user_data: {get_attr: [boot_config, config]}

outputs:

  server_ip:
    value: { get_attr: [ server, addresses, public, 0, addr ] }
    description: IP address of the server

  admin_password:
    value: { get_attr: [ admin_password, value ] }
    description: Root password to the server

The Heat::InstallConfigAgent Resource
=====================================

You will notice that this resource has no real properties or other configuration. That's
because we use the Environment and Template Resource features of Heat so that we can
create several bootstrap configurations and use them for different base images as
required.

The Configuration Template
--------------------------

First, lets look at the template that we'll use to provide the underlying definition for
the `boot_config` resource. Since this template is a bit large, it won't be included in
its entirety here, but can always be found in the `templates` directory of this
repository as `bootconfig_all.yaml`.

In _Generic Software Config_, we used the same mechanism to bootstrap our clean instance
using a template provided by the OpenStack Heat project. While that works well, the
repository used is laid out for maximum reusability, so it can be hard to follow what's
actually going on in the template. For this tutorial, we've "de-normalized" the bootstrap
template to more easily explain the different sections and what they do.

Before we dive in, also note that there isn't anything special about this template. Heat
allows for and encourages template composition so that you can abstract and re-use parts
of your application architecture. Having said that, we won't talk at all about basic
things like descriptions or versions but rather go over the resources and how they
prepare the instance for use with Heat Software Config.

Install the Basics
++++++++++++++++++

The first resource is the most complex and uses cloud-init to lay down the needed
software, scripts, and configuration needed. Since this there is a lot going on here,
we'll break down the actual cloud-config rather than the resource wrapping it.

First, we install the supporting software packages:

..code:: yaml

        apt_upgrade: true
        apt-sources:
        - source: "ppa:ansible/ansible"
        packages:
        - python-pip
        - git
        - gcc
        - python-dev
        - libyaml-dev
        - libssl-dev
        - libffi-dev
        - libxml2-dev
        - libxslt1-dev
        - python-apt
        - ansible
        - salt-minion

The next section writes several files. The first four are fairly generic and are to
configure the base OpenStack agents `os-collect-config`, `os-apply-config`, and
`os-refresh-config`. Note that these agents are actually installed in a separate section
described later. You can read more about these agents in the reference sections. Their job
is to coordinate the reading, running, and updating of the software configuration that
will be sent via Heat.

Following few files tell the generic OpenStack agents how to handle configurations
received from Heat. The script written to
`/opt/stack/os-config-refresh/configure.d/55-heat-config` is executed when a config is to
be applied or refreshed. It is this script that decides which config handler agent to call
to apply the configuration (shell script, Ansible, Puppet, Salt, etc).

The script written to `/var/lib/heat-config/hooks/script` is the default config handler
agent that executes configuration in the `default` group and assumes the configuration is
a shell script.

The other available agent handlers are written similarly using the same root hooks
directory (`/var/lib/heat-config/hooks`) and using the name of the config group handled as
the file name. In our example, we've included handlers for using configurations in the
default, Ansible, Salt, and Puppet config groups. You can customize this for your needs by
removing handlers you don't want or adding additional ones from
`<https://github.com/openstack/heat-templates/tree/master/hot/software-config/elements>`_.
Note that you may also need to add required packages to the `packages` or `runcmd`
sections of the cloud-config if you add additional handlers.

The final section installs puppet for the puppet group handler and the runs the commands
that bootstrap the generic OpenStack agents.

..code:: yaml

        runcmd:
        - wget https://apt.puppetlabs.com/puppetlabs-release-trusty.deb
        - dpkg -i puppetlabs-release-trusty.deb
        - apt-get update
        - apt-get install puppet
        - os-collect-config --one-time --debug
        - cat /etc/os-collect-config.conf
        - os-collect-config --one-time --debug

Install the Generic Agents
++++++++++++++++++++++++++

The actual generic OpenStack agents are installed using Python pip since there aren't any
reliable packages for them on Ubuntu.

..code:: yaml

  install_agents:
    type: "OS::Heat::SoftwareConfig"
    properties:
      group: ungrouped
      config: |
        #!/bin/bash
        set -eux
        pip install os-collect-config os-apply-config os-refresh-config dib-utils

Configure the Agents Service
++++++++++++++++++++++++++++

Next, we declare a config resource to create the service configuration (upstart or
systemd) that will start the collection agent and ensure that it runs on boot:

..code:: yaml

  start:
    type: "OS::Heat::SoftwareConfig"
    properties:
      group: ungrouped
      config: |
        #!/bin/bash
        set -eux

        if [[ `systemctl` =~ -\.mount ]]; then

            # if there is no system unit file, install a local unit
            if [ ! -f /usr/lib/systemd/system/os-collect-config.service ]; then

                cat <<EOF >/etc/systemd/system/os-collect-config.service
        [Unit]
        Description=Collect metadata and run hook commands.

        [Service]
        ExecStart=/usr/bin/os-collect-config
        Restart=on-failure

        [Install]
        WantedBy=multi-user.target
        EOF

        cat <<EOF >/etc/os-collect-config.conf
        [DEFAULT]
        command=os-refresh-config
        EOF
            fi

            # enable and start service to poll for deployment changes
            systemctl enable os-collect-config
            systemctl start --no-block os-collect-config
        elif [[ `/sbin/init --version` =~ upstart ]]; then
            if [ ! -f /etc/init/os-collect-config.conf ]; then

                cat <<EOF >/etc/init/os-collect-config.conf
        start on runlevel [2345]
        stop on runlevel [016]
        respawn

        # We're logging to syslog
        console none

        exec os-collect-config  2>&1 | logger -t os-collect-config
        EOF
            fi
            initctl reload-configuration
            service os-collect-config start
        else
            echo "ERROR: only systemd or upstart supported" 1>&2
            exit 1
        fi

Combine and expose the Configs
++++++++++++++++++++++++++++++

Finally, the configurations are all combined into a single multi-part-mime so that they 
can be output as a single file for use in user-data:

..code:: yaml

  install_config_agent:
    type: "OS::Heat::MultipartMime"
    properties:
      parts:
      - config: { get_resource: configure }
      - config: { get_resource: install_agents }
      - config: { get_resource: start }

..code:: yaml

outputs:
  config:
    value: { get_resource: install_config_agent }

The Environment File
--------------------

The environment file that we'll send as part of our `stack-create` call is quite simple:

..code:: yaml

# Installs software-config agents for ubuntu with pip install

parameters:
  image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)

resource_registry:
  "Heat::InstallConfigAgent": bootconfig_all.yaml

This sets the `image` parameter value to "Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)" and maps
the resource namespace `Heat::InstallConfigAgent` to the template resource we created in
the previous section. If you've used another file name or want to use the one included in
this repository, be sure to change this mapping to point to the appropriate location.

Deploy the Bootstrapped Instance
================================

All that's left to do is deploy the template:

..code:: example

 heat stack-create -f templates/software_config_custom_image.yaml -e templates/bootconfig.all.env.yaml sw_config_base

Wait for the stack to be `CREATE_COMPLETE` an you have a basic vm configured for use
with Heat software config. You can stop here and modify this template to actually deploy
software configurations to your server using `OS::Heat::SoftwareConfig` and
`OS::Heat::SoftwareDeployment` using "clean" images. However, the next section explains
how you can use this bootstrapped instance to create your own image pre-configured for use
with Heat software config. However, future advanced tutorials such as using Heat with
Ansible will make use of this pre-bootstrapped image so you  may want to continue with
the next section anyway.

Custom Image
============

Remove Cloud-Init Artifacts
---------------------------

In order for cloud-init to run on machines booted from our new image, we'll need to
remove some artifacts from the current vm left over from our initial bootstrapping. First,
retrieve the root password from the stack:

..code: example

heat output-show sw_config_base admin_password

Now, log into the server via ssh by issuing the following command:

..code: example

ssh root@$(heat output-show sw_config_base server_ip)

Enter the password you retrieved previously.

Once logged into the server, run the following commands to remove the artifacts created by
cloud-init when it bootstrapped this server:

- :bash:`rm /var/lib/cloud/instance`
- :bash:`rm -rf /var/lib/cloud/instances/*`
- :bash:`rm -rf /var/lib/cloud/data/*`
- :bash:`rm /var/lib/cloud/sem/config_scripts_per_once.once`
- :bash:`rm /var/log/cloud-init.log`
- :bash:`rm /var/log/cloud-init-output.log`

Snapshot Your Bootstrapped Server
---------------------------------

Now we can create an image of our server. First, log into the Reach control panel and
under Orchestration, find the 'sw_config_base` stack. Viewing the details, you should see
the server listed in the `Infrastructure` section. Select that server to view its details.
Under the `Actions` button, select `Create an Image` and name it "Ubuntu 14.04 LTS (HEAT)".

Once this process is complete, you're all done!

Using Your New Image
--------------------

We will make use of this new image in our future tutorials on using Heat software config,
but in short, you can omit using the `Heat::InstallConfigAgent` resource once you have
this image. Instad, set the `image` property of any servers you want to configure this way
to "Ubuntu 14.04 LTS (HEAT)" and the `user_data_format` property to "SOFTWARE_CONFIG" and
it should just work!

Reference documentation
=======================

- `<http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareConfig][OS::Heat::SoftwareConfig>`_
- `<http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Heat::SoftwareDeployment][OS::Heat::SoftwareDeployment>`_
- `<http://docs.openstack.org/developer/heat/template_guide/composition.html>`_
- `<http://docs.openstack.org/developer/heat/template_guide/environment.html>`_
- `<https://github.com/openstack/os-collect-config>`_
- `<https://github.com/openstack/os-refresh-config>`_
- `<https://github.com/openstack/os-apply-config>`_
