===================================
 SwiftSignal and SwiftSignalHandle
===================================

Brief summary
=============

SwiftSingal can be used to coordinate resource creation with
notifications/signals that could be coming from sources external or
internal to the stack. It is often used in conjunction with
SwiftSignalHandle resource.

SwiftSignalHandle is used to create a temporary URL and this URL is used
by applications/scripts to send signals. SwiftSignal resource waits on
this URL for a specified number of signals in given time.

Example template
================

In the following example template, we will set up a single node linux
server that signals success/failure of user_data script
execution at a given URL.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Single node linux server with swift signaling.

    resources:

    outputs:

Resources section
-----------------

Add a SwiftSignalHandle resource
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SwiftSignalHandle is a resource to create temporary URL to receive
notification/signals. Note that the temporary URL is created using Rackspace
Cloud Files.

.. code:: yaml

      signal_handle:
        type: "OS::Heat::SwiftSignalHandle"

Add SwiftSignal resource
~~~~~~~~~~~~~~~~~~~~~~~~

The SwiftSignal resource waits for specified number of "SUCCESS" signals (number
is provided as 'count' property) on the given URL ('handle' property).
The stack will be marked as failure if specificed number of signals are
not received in given timeout or if a non "SUCCESS" signal is received such as a "FAILURE". A data string and a reason string may be attached alongwith the success or failure notification. The data string is an attribute that can be pulled as template output. 

.. code:: yaml

      wait_on_server:
        type: OS::Heat::SwiftSignal
        properties:
          handle: {get_resource: signal_handle}
          count: 1
          timeout: 600

Here SwiftSignal resource would wait for 600 seconds to receive 1 signal
on the handle.

Add a Server resource
~~~~~~~~~~~~~~~~~~~~~

Add a linux server with a bash script in user_data property. At
the end of the script execution send a success/failure message to the
temporary URL created by the above SwiftSignalHandle resource.

.. code:: yaml

      linux_server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance
          user_data:
            str_replace:
              template: |
                #!/bin/bash -x
                # assume you are doing a long running operation here
                sleep 300

                # Assuming long running operation completed successfully, notify success signal
                wc_notify --data-binary '{"status": "SUCCESS", "data": "Script execution succeeded"}'
                
                # Alternatively if operation fails a FAILURE with reason and data may be sent,
                # notify failure signal example below
                # wc_notify --data-binary '{"status": "FAILURE", "reason":"Operation failed due to xyz error", "data":"Script execution failed"}'
                
              params:
                # Replace all occurances of "wc_notify" in the script with an
                # appropriate curl PUT request using the "curl_cli" attribute
                # of the SwiftSignalHandle resource
                wc_notify: { get_attr: ['signal_handle', 'curl_cli']

Outputs section
---------------

Add swift signal URL to the outputs section.

.. code:: yaml

      #Get the signal URL which contains all information passed to the signal handle
      signal_url:
        value: { get_attr: ['signal_handle', 'curl_cli'] }
        description: Swift signal URL
      
      #Obtain data describing script results. If nothing is passed, this value will be NULL 
      signal_data:
        value: { get_attr: ['wait_on_server', 'data'] }
        description: Data describing script results
        
      server_public_ip:
        value:{ get_attr: [ linux_server, accessIPv4 ] }
        description: Linux server public IP

Full Example Template
---------------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Single node linux server with swift signaling.

    resources:

      signal_handle:
        type: "OS::Heat::SwiftSignalHandle"
        
      wait_on_server:
        type: OS::Heat::SwiftSignal
        properties:
          handle: {get_resource: signal_handle}
          count: 1
          timeout: 600
          
      linux_server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance
          user_data:
            str_replace:
              template: |
                #!/bin/bash -x
                # assume you are doing a long running operation here
                sleep 300

                # Assuming long running operation completed successfully, notify success signal
                wc_notify --data-binary '{"status": "SUCCESS", "data": "Script execution succeeded"}'
                
                # Alternatively if operation fails a FAILURE with reason and data may be sent,
                # notify failure signal example below
                # wc_notify --data-binary '{"status": "FAILURE", "reason":"Operation failed due to xyz error", "data":"Script execution failed"}'

              params:
                wc_notify: { get_attr: ['signal_handle', 'curl_cli'] }

    outputs:
      #Get the signal URL which contains all information passed to the signal handle
      signal_url:
        value: { get_attr: ['signal_handle', 'curl_cli'] }
        description: Swift signal URL
      
      #Obtain data describing script results. If nothing is passed, this value will be NULL 
      signal_data:
        value: { get_attr: ['wait_on_server', 'data'] }
        description: Data describing script results

      server_public_ip:
        value: { get_attr: [ linux_server, accessIPv4 ] }
        description: Linux server public IP

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/overview.html>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud-init format
   documentation <http://cloudinit.readthedocs.org/en/latest/topics/format.html>`__
-  `Swift
   TempURL <http://docs.rackspace.com/files/api/v1/cf-devguide/content/TempURL-d1a4450.html>`__
