=========================
 Rackspace CDN using Heat
=========================

Brief summary
=============

Rackspace CDN gives you the power to accelerate content on any public resource at
Rackspace. It provides a simple API and Control Panel experience for you to manage your
CDN-enabled domains and the origins and assets associated with those domains. You can use
Rackspace Cloud Orchestration to simplify content deployment and CDN configuration.

Note that Rackspace CDN is a separate service and is different from Rackspace Cloud Files
CDN capability. Rackspace Cloud Orchestration supports both options, but this document
focuses only on the Rackspace CDN service. Using Cloud Files CDN with Cloud Orchestration
is described in another document.

You can download the full template for this example from `this repository's
templates directory <https://github.com/rackerlabs/rs-heat-docs/blob/master/templates/raxcdn.yaml>`_.

Prerequisite(s):
================

You should be familiar with general Heat template authoring and resource usage. You should
also be familiar with Rackspace CDN and its configuration:

- `Rackspace Cloud Orchestration Templates Developer Guide 
  <https://docs.rackspace.com/orchestration/api/v1/orchestration-templates-devguide/content/overview.html>`_
- `Rackspace CDN Getting Started Guide
  <https://developer.rackspace.com/docs/cdn/getting-started>`_

Example Template
================

This is a simple template that will illustrate using the `Rackspace::Cloud::CDN` resource
to enable CDN distribution of content served from a Cloud Server.

As always, we start with a basic template outline:

.. code:: yaml

  heat_template_version: 2015-10-15

  description: |
    Template for testing Rackspace CDN resource

  resources:

  outputs:

Resources section
-----------------

First, we add some signaling resources so that we can let Cloud Orchestration know
when our content server has been configured:

.. code:: yaml

  resources:

    wait_on_content:
      type: OS::Heat::SwiftSignal
      properties:
        handle: { get_resource: wait_handle }
        timeout: 600

    wait_handle:
      type: OS::Heat::SwiftSignalHandle


We will also need to register some records with Cloud DNS, so lets generate a random
domain name so that we don't accidentally try and create a domain we've already used:

.. code:: yaml

    random_domain:
      type: OS::Heat::RandomString
      properties:
        length: 13
        character_classes:
        - class: letters

For configuring the content server, we'll specify our initialization separately, just to
keep the template resources clean and easily modifiable:

.. code:: yaml

    content_config:
      type: OS::Heat::CloudConfig
      properties:
        cloud_config:
          packages:
          - apache2
          write_files:
          - path: /root/phonehome.sh
            permissions: "0700"
            content:
              str_replace:
                template: |
                  #!/bin/bash
                  wc_notify --data-binary '{"status": "SUCCESS"}'
                params:
                  wc_notify: { get_attr: [ 'wait_handle', 'curl_cli' ] }
          runcmd:
          - /root/phonehome.sh

The `OS::Heat::CloudConfig` resource simply allows us to specify our cloud config file
in yaml in the template rather than in a string in the content server's `user_data`
property. This configuration is pretty simple in that all it does is install the Apache
web server, create a script for calling our notification hook, and then calls that hook.
You can imagine doing more sophisticated configuration that could include pulling
site content from a repository or optimizing the web server configuration. For this
example, however, we'll keep it simple and install a common web server with default
content.

Now we need to define the content server itself:

.. code:: yaml

    content:
      type: OS::Nova::Server
      properties:
        name:
          str_replace:
            template: stack-content
            params:
              stack: { get_param: "OS::stack_name" }
        metadata:
          rax-heat: { get_param: "OS::stack_id" }
        image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
        flavor: 1 GB Performance
        user_data_format: RAW
        user_data: { get_attr: [ content_config, config ] }

Notice that we use the ``content_config`` resource to provide the ``user_data`` as
described earlier.

In order for Rackspace CDN to serve our content correctly and for our users to have
access from our site, we'll need to create a DNS CNAME record for our domain:

.. code:: yaml

    domain:
      type: Rackspace::Cloud::DNS
      properties:
        name:
          str_replace:
            template: domain.com
            params:
              domain: { get_param: "OS::stack_name" }
        emailAddress: heat@lists.rackspace.com
        records:
        - name:
            str_replace:
              template: www.domain.com
              params:
                domain: { get_param: "OS::stack_name" }
          type: CNAME
          data: { get_attr: [ cdn, access_url ] }

Notice that we'll need some information from the Rackspace CDN resource before we can
create the right CNAME record. We'll define that resource in a minute, but notice that
with Cloud Orchestration, the order of our resources doesn't really matter. Cloud
Orchestration "knows" that the DNS resource needs some information from the CDN resource
and won't do things out of order.

Finally, lets create the Rackspace CDN service that will distribute our content from our
Cloud Server:

.. code:: yaml

    cdn:
      type: Rackspace::Cloud::CDN
      depends_on: wait_on_content
      properties:
        name:
          str_replace:
            template: stack-cdn
            params:
              stack: { get_param: "OS::stack_name" }
        domains:
        - domain:
            str_replace:
              template: www.domain.com
              params:
                domain: { get_param: "OS::stack_name" }
        origins:
        - origin: { get_attr: [content, accessIPv4] }
        caching:
        - name: default
          ttl: 360
        flavor_id: cdn

Here, we've asked the Rackspace CDN service to distribute content from our Cloud Server
(the ``origins`` property) for the domain (the ``domains`` property) we configured 
previously.

Outputs section
---------------

We will use the ``outputs`` section to get relevant information from the CDN configuration:

.. code:: yaml

  outputs:

    server_ip:
      description: IP address of the content server
      value: { get_attr: [content, accessIPv4] }

    cdn_id:
      description: ID of the CDN service
      value: { get_resource: cdn }

    cdn_access_url:
      description: Access URL for cdn resources
      value: { get_attr: [ cdn, access_url ] }

    cdn_log_url:
      description: Log URL for cdn resource (should be empty!)
      value: { get_attr: [ cdn, log_url ] }


Full example template
---------------------

You can see the full template at `<https://github.com/rackerlabs/rs-heat-docs/blob/master/templates/raxcdn.yaml>`_.

Reference
=========

- `Rackspace CDN Developer Guide
  <https://developer.rackspace.com/docs/cdn/v1/developer-guide/>`_
- `Rackspace::Cloud::CloudFilesCDN Resource Documentation
  <http://orchestration.rackspace.com/raxdox/rackspace.html#Rackspace::Cloud::CDN>`_
