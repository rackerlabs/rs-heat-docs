==========================
 Auto-scaling Heat stacks
==========================

Brief summary
=============

In the *Event and schedule-based auto-scaling with Cloud
Orchestration* tutorial, we used the "server" launchConfiguration type
to scale the Cloud Servers in your web application.  Rackspace Auto
Scale also supports a "stack" launchConfiguration type, where the unit
of scale is a stack instead of a server.

In this tutorial, we will learn how to scale using a group of Heat
stacks which allows us to scale configurations more complex than a
single server. In this example, we will use a stack containing both a
server and a ``OS::Heat::SoftwareConfig`` to configure new instances
in the group.


Pre-reading
===========

The *Auto Scale* and *Cloud Orchestration* pre-reading from the *Event
and schedule-based auto-scaling with Cloud Orchestration* tutorial
will be useful in this tutorial as well.  In addition, please look
over the new `"launchConfiguration.args" body parameters
<https://github.com/dragorosson/otter/blob/991a367c44fe9623bb5d0a7b2ae24fe0205d0c33/api-docs/rst/dev-guide/api-operations/methods/post-create-scaling-group-v1.0-tenantid-groups.rst>`__
in the Otter docs.


Example template
================

Start by creating a new template with the following top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and Event-based scaling using Rackspace Cloud Autoscale

    parameters:

    resources:

Add a parameter for ``key_name``, ``flavor``, and ``image``:

.. code:: yaml

    key_name:
      type: string
      description : Name of a key pair to enable SSH access to instances.
      default: my_key

    flavor:
      type: string
      description: Flavor to use for the WordPress server.
      constraints:
        - custom_constraint: nova.flavor
      default: 4 GB Performance

    image:
      type: string
      description: >
        Name or ID of the image to use for the WordPress server.
        The image must have the software config agents baked-in.
      default: f4bbbce2-50b0-4b07-bf09-96c175a45f4b

It is important that the image being used has the the base OpenStack
agents (``os-collect-config``, ``os-apply-config``, and
``os-refresh-config``) baked into the image.

Next, add a ``Rackspace::AutoScale::Group`` resource with a
 ``launchConfiguration`` type of ``launch_stack``:

.. code:: yaml

    lamp_asg:
      type: Rackspace::AutoScale::Group
      properties:
        groupConfiguration:
          name: { get_param: "OS::stack_name" }
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
          maxEntities: 3
          minEntities: 1
          cooldown: 120
        launchConfiguration:
          type: launch_stack
          args:
            stack:
              template_url: https://raw.githubusercontent.com/rackerlabs/rs-heat-docs/master/templates/launch_stack_template.yaml
              disable_rollback: False
              parameters:
                flavor: {get_param: flavor}
                image: {get_param: image}
                key_name: {get_param: key_name}
              timeout_mins: 30

The template referenced by URL in the ``template_url`` property is the
template for the stack being scaled.  It is a simple template that
creates a LAMP server using an ``OS::Nova::Server``,
``OS::Heat::SoftwareDeployment``, and ``OS::Heat::SoftwareConfig``
resource.  Please read through `the template
<https://raw.githubusercontent.com/rackerlabs/rs-heat-docs/master/templates/launch_stack_template.yaml>`__
before proceeding.

Next, add Rackspace::AutoScale::ScalingPolicy and
Rackspace::AutoScale::WebHook resources to create webhooks for scaling
up and down:

.. code:: yaml

  scale_up_policy:
    type: Rackspace::AutoScale::ScalingPolicy
    properties:
      group: { get_resource: lamp_asg }
      name:
        str_replace:
          template: stack scale up policy
          params:
            stack: { get_param: "OS::stack_name" }
      change: 1
      cooldown: 600
      type: webhook

  scale_up_webhook:
    type: Rackspace::AutoScale::WebHook
    properties:
      name:
        str_replace:
          template: stack scale up hook
          params:
            stack: { get_param: "OS::stack_name" }
      policy: { get_resource: scale_up_policy }
  
  scale_down_policy:
    type: Rackspace::AutoScale::ScalingPolicy
    properties:
      group: { get_resource: lamp_asg }
      name:
        str_replace:
          template: stack scale down policy
          params:
            stack: { get_param: "OS::stack_name" }
      change: -1
      cooldown: 600
      type: webhook

  scale_down_webhook:
    type: Rackspace::AutoScale::WebHook
    properties:
      name:
        str_replace:
          template: stack scale down hook
          params:
            stack: { get_param: "OS::stack_name" }
      policy: { get_resource: scale_down_policy }

Finally, add following to the outputs section so that the webhooks
created above are displayed in the stack outputs:

.. code:: yaml

  "Scale UP webhook":
    value: { get_attr: [ scale_up_webhook, executeUrl ] }
  
  "Scale DOWN webhook":
    value: { get_attr: [ scale_down_webhook, executeUrl ] }

The full template can be found at
https://raw.githubusercontent.com/rackerlabs/rs-heat-docs/master/templates/launch_stack.yaml

Create stack and scale group
============================

Create the parent stack with the following heatclient command:

.. code:: bash

    heat stack-create -f https://raw.githubusercontent.com/rackerlabs/rs-heat-docs/master/templates/launch_stack.yaml launch_stack

The scaling group will be created within a few seconds (the stack will
show as ``CREATE COMPLETE``) and then the scaling group will begin
scaling to the specified minimum number of entities (one stack, in our
case).  In a few minutes, the PHP info page should become available at
the URL shown in the stack details page in the MyRackspace Portal for
the child stack that was created by AutoScale.

To see the AutoScale group scale, view the "Scale UP webhook" in the
stack outputs and then trigger the webhook using curl:

.. code:: bash

    curl -X POST -I -k <webhook_scale_up_address_from_stack_outputs>

You should now have two LAMP stacks and associated LAMP servers.
