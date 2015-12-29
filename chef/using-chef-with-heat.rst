.. _using_chef_w_heat:

====================
Using Chef with Heat
====================

Brief summary
=============

In this tutorial, we will show you how to leverage Chef via Heat and software config to
bootstrap an instance with a fully configured Redis server.

Pre-reading
===========

- You should already be familiar with :ref:`Generic Software Config <generic-software-config>`
- You should already be familiar with `Chef <https://www.chef.io/>`_, its usage and features.

Basic template
==============

As with all Heat templates, we start with the basic version and description sections:

.. code:: yaml

  heat_template_version: 2014-10-16 

  description: |
    Using Orchestration software config and Chef

There will be no parameters or outputs in this example, but you can add them as well as
appropriate calls to intrinsic functions (``get_attr``, ``get_resource``, etc) if you
want to make the example more configurable.

Resources
---------

The first resource we'll add is the server we want to configure with Chef:

.. code:: yaml

    server:
      type: OS::Nova::Server
      properties:
        name:
          str_replace:
            template: stack-server
            params:
              stack: { get_param: "OS::stack_name" } 
        metadata:
          rax-heat: { get_param: "OS::stack_id" }
        image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM) (Orchestration)
        flavor: 2 GB Performance
        config_drive: true
        software_config_transport: POLL_TEMP_URL
        user_data_format: SOFTWARE_CONFIG

This server uses an image pre-configured with all of the agents needed to run software
configuration. You can also use your own custom image or bootstrap a "pristine" image
in your template. See :ref:`Bootstrapping Software Config <bootstrapping_sw_config>` if
you are unfamiliar with that process.

Also note that we've set up ``config_drive``, ``software_config_transport``
and ``user_data_format`` as required.

Next we specify a random password to use when accessing the Redis service on the instance:

.. code:: yaml

  redis_password:
    type: OS::Heat::RandomString
    properties:
      length: 16
      sequence: lettersdigits

We then use an ``OS::Heat::SoftwareConfig`` to define what attributes, recipes, and
repository the Chef software config agent should apply:

.. code:: yaml

  redis_config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: chef
      inputs:
      - name: redisio
        type: Json
      config: |
        ["recipe[apt]",
         "recipe[build-essential]",
         "recipe[redisio::default]",
         "recipe[redisio::enable]" ]
      options:
        kitchen: https://github.com/rackspace-orchestration-templates/redis-single
        kitchen_path: /opt/heat/chef/kitchen

For this agent, use the group ``chef`` to tell Orchestration which agent should process
the configuration.

For this agent, it is important to specify the top-level elements of any attribute
overrides you plan to use in the ``inputs`` section to ensure that this information is
formatted correctly when sent to the agent.

The ``config`` property simply defines the run-list you want applied to the instance.
Additionally, the chef agent allows for an input named ``environment`` of type `String`
that you can use to specify which environment to use when applying the config. You do not
have to explicitly declare this input in the config resource. We don't use this input in
this example, but it is included in a comment in a following section to illustrate its
use.

The ``options`` property allows you to optionally specify both the source location and the
local path to the kitchen containing the roles, recipes, attributes, and other elements
needed to converge the instance. The ``kitchen_path`` property defaults to
``/var/lib/heat-config/heat-config-chef/kitchen`` if it is not specified.

``kitchen`` allows you to specify the url of a Github repository that contains your
kitchen. Here, we re-use a repository from one of the existing Rackspace curated
templates. If you do not specify a ``kitchen`` to clone, you will need to make sure that
your kitchen is available at the specified ``kitchen_path`` either via another
``OS::Heat::SoftwareConfig`` resource, ``user_data``, custom image, or some other "manual"
means.


Finally we deploy the configuration to the instance:

.. code:: yaml

  deploy_redis:
    type: OS::Heat::SoftwareDeployment
    properties:
      signal_transport: TEMP_URL_SIGNAL
      input_values:
        # environment: production -- This isn't used in this example
        redisio:
          default_settings:
            requirepass: { get_attr: [redis_password, value] }
          servers:
          - name:
              str_replace:
                template: stack-server
                params:
                  stack: { get_param: "OS::stack_name" } 
            port: 6379
          version: "2.8.14"
      config:
        get_resource: redis_config
      server:
        get_resource: server

Note that the input values take the form of a dictionary just like they would for any
other node. Also note as mentioned earlier that we've commented out the ``environment``
input since its not actually used in the recipes we've used.

References
==========

- `Full template for this example <https://github.com/rackerlabs/rs-heat-docs/blob/master/chef/templates/chefsetup.yaml>`_
- `Kitchen used in this example <https://github.com/rackspace-orchestration-templates/redis-single>`_
