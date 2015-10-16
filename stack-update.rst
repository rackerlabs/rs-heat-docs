=================
 Updating Stacks
=================

Overview
========

Rackspace Orchestration has the ability to modify running stacks using
the `update stack operation
<http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/PUT_stack_update__v1__tenant_id__stacks__stack_name___stack_id__Stack_Operations.html>`__.
This gives you the ability to add, edit, or delete resources in a
stack.  In the python-heatclient CLI, this operation is called `heat
stack-update`.

Create a stack
==============

For this tutorial, we will create a simple stack with one server:

.. code:: yaml

    heat_template_version: 2013-05-23
    
    resources:
      hello_world:
        type: "OS::Nova::Server"
        properties:
          flavor: 1GB Standard Instance
          image: 5b0d5891-f80c-412a-9b73-cc996de9d719
          config_drive: "true"
          user_data_format: RAW
          user_data: |
            #!/bin/bash -xv
            echo "hello world" > /root/hello-world.txt
    outputs:
      public_ip:
        value: { get_attr: [ hello_world, accessIPv4 ] }
        description: The public ip address of the server

Save this template as `stack-update-example.yaml` and create a stack
using the following heatclient command:

.. code:: example

    heat stack-create -f stack-update-example.yaml stack-update-example

Update the stack without replacement
====================================

Next, edit the template file and change the server flavor to "2 GB
Standard Instance":

.. code:: yaml

    heat_template_version: 2013-05-23
    
    resources:
      hello_world:
        type: "OS::Nova::Server"
        properties:
          flavor: 2GB Standard Instance
          image: 5b0d5891-f80c-412a-9b73-cc996de9d719
          config_drive: "true"
          user_data_format: RAW
          user_data: |
            #!/bin/bash -xv
            echo "hello world" > /root/hello-world.txt
    outputs:
      public_ip:
        value: { get_attr: [ hello_world, accessIPv4 ] }
        description: The public ip address of the server

Modifying some resource properties will trigger a delete->rebuild of
that resource.  Because some architectures are less tolerant of nodes
being rebuilt, you can check the `Template Guide
<http://docs.openstack.org/developer/heat/template_guide/index.html>`__
to see which properties trigger a rebuild.  For example, each property
in the `OS::Nova::Server documentation
<http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Nova::Server>`__
says either "Updates cause replacement" or "Can be updated without
replacement".

Alternatively, you can preview what will happen with the stack is
updated by adding a "-y" option to the "heat stack-update" command:

.. code:: example

    heat stack-update -y -f stack-update-example.yaml stack-update-example

The `hello_world` resource should show up in the "updated" section,
since resizing can be done without replacement.

To actually update the stack, resubmit the modified template:

.. code:: example

    heat stack-update -f stack-update-example.yaml stack-update-example

If there were any parameters or flags passed to the original
stack-create, they need to be passed unmodified to the stack-update
command (unless you are changing them as part of the stack-update).
Leaving them out may result in unexpected changes to the stack.

Update the stack with replacement
=================================

In the next example, we will modify a property that will cause the
server to be rebuilt.  Change "hello world" to "foo" in the
`user_data` section:

.. code:: yaml

    heat_template_version: 2013-05-23
    
    resources:
      hello_world:
        type: "OS::Nova::Server"
        properties:
          flavor: 2GB Standard Instance
          image: 5b0d5891-f80c-412a-9b73-cc996de9d719
          config_drive: "true"
          user_data_format: RAW
          user_data: |
            #!/bin/bash -xv
            echo "foo" > /root/hello-world.txt
    outputs:
      public_ip:
        value: { get_attr: [ hello_world, accessIPv4 ] }
        description: The public ip address of the server

The stack-update preview output with this template should result in
the `hello_world` resource being in the "replaced" section:

.. code:: example

    heat stack-update -y -f stack-update-example.yaml stack-update-example

Issue the update as before:

.. code:: example

    heat stack-update -f stack-update-example.yaml stack-update-example

Update the stack to add a resource
==================================

In this example, we will add a resource to a stack.  Add another
server to the template:

.. code:: yaml

    heat_template_version: 2013-05-23
    
    resources:
      hello_world:
        type: "OS::Nova::Server"
        properties:
          flavor: 2GB Standard Instance
          image: 5b0d5891-f80c-412a-9b73-cc996de9d719
          config_drive: "true"
          user_data_format: RAW
          user_data: |
            #!/bin/bash -xv
            echo "foo" > /root/hello-world.txt

      hello_world2:
        type: "OS::Nova::Server"
        properties:
          flavor: 2GB Standard Instance
          image: 5b0d5891-f80c-412a-9b73-cc996de9d719
          config_drive: "true"
          user_data_format: RAW
          user_data: |
            #!/bin/bash -xv
            echo "bar" > /root/hello-world.txt

    outputs:
      public_ip:
        value: { get_attr: [ hello_world, accessIPv4 ] }
        description: The public ip address of the server
      public_ip2:
        value: { get_attr: [ hello_world2, accessIPv4 ] }
        description: The public ip address of the server

The stack-update preview output with this template should result in
the `hello_world2` resource being in the "added" section, and the
`hello_world` resource being in the "unchanged" section:

.. code:: example

    heat stack-update -y -f stack-update-example.yaml stack-update-example

Issue the update to create the other server:

.. code:: example

    heat stack-update -f stack-update-example.yaml stack-update-example

