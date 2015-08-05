=====================================
 Rackspace Cloud Files CDN using Heat
=====================================

Brief summary
=============

A CDN-enabled container is a public container that is served by the Akamai
content delivery network. The files in a CDN-enabled container are publicly
accessible and do not require an authentication token for read access.
However, uploading content into a CDN-enabled container is a secure operation
and does require a valid authentication token. (Private containers are not
CDN-enabled and the files in a private container are not publicly accessible.)

You can download the full template for this example from this repository's
`templates` directory.

Prerequisite(s):
================

You should be familiar with general Heat template authoring and resource usage.

Example Template
================

This is a simple template that will illustrate using the
`Rackspace::Cloud::CloudFilesCDN` resource to enable CDN functionality on a Cloud Files
container.

As always, we start with a basic template outline:

.. code:: yaml

  heat_template_version: 2015-10-15

  description: |
    Test Cloud Files CDN

  resources:

  outputs:

Resources section
-----------------

We only need two simple resources for this template:

.. code:: yaml

  resources:

    container:
      type: OS::Swift::Container
      properties:
        name: { get_param: "OS::stack_name" }

    container_cdn:
      type: Rackspace::Cloud::CloudFilesCDN
      properties:
        container: { get_resource: container }
        ttl: 3600

The `container` resource simply creates a new Cloud Files container while the
`container_cdn` resource activates CDN functionality for that container. The
`container` property defines the container to enable while the `ttl` property tells
the CDN service how long to cache objects.

Outputs section
---------------

We'll use the outputs section to get relavent information from the CDN configuration:

.. code:: yaml

  outputs:

    show:
      value: { get_attr: [ container_cdn, show ] }
      description: Show all attributes of the CDN configuration for the container

    cdn_url:
      value: { get_resource: container_cdn }
      description: The URI for downloading container objects over HTTP.

    ssl_url:
      value: { get_attr: [ container_cdn, ssl_uri ] }
      description: The URI for downloading container objects over HTTPS, using SSL.

    streaming_url:
      value: { get_attr: [ container_cdn, streaming_uri ] }
      description: |
        The URI for video streaming container objects that uses HTTP Dynamic
        Streaming from Adobe.

    ios_url:
      value: { get_attr: [ container_cdn, ios_uri ] }
      description: |
        The URI for video streaming container objects that uses HTTP Live
        Streaming from Apple.


Full Example Template
---------------------

.. code:: yaml

  heat_template_version: 2015-10-15

  description: |
    Test Cloud Files CDN

  resources:

    container:
      type: OS::Swift::Container
      properties:
        name: { get_param: "OS::stack_name" }

    container_cdn:
      type: Rackspace::Cloud::CloudFilesCDN
      properties:
        container: { get_resource: container }
        ttl: 3600

  outputs:

    show:
      value: { get_attr: [ container_cdn, show ] }
      description: Show all attributes of the CDN configuration for the container

    cdn_url:
      value: { get_resource: container_cdn }
      description: The URI for downloading container objects over HTTP.

    ssl_url:
      value: { get_attr: [ container_cdn, ssl_uri ] }
      description: The URI for downloading container objects over HTTPS, using SSL.

    streaming_url:
      value: { get_attr: [ container_cdn, streaming_uri ] }
      description: |
        The URI for video streaming container objects that uses HTTP Dynamic
        Streaming from Adobe.

    ios_url:
      value: { get_attr: [ container_cdn, ios_uri ] }
      description: |
        The URI for video streaming container objects that uses HTTP Live
        Streaming from Apple.

Reference
=========

- `Cloud Files CDN API Documentation
  <http://docs.rackspace.com/files/api/v1/cf-devguide/content/API_Operations_for_CDN_Services-d1e2386.html>`_
- `Rackspace::Cloud::CloudFilesCDN Resource Documentation
  <http://orchestration.rackspace.com/raxdox/rackspace.html#Rackspace::Cloud::CloudFilesCDN>`_
