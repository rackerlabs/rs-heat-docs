================================================================
 Event and schedule-based auto scaling with Heat
================================================================

Brief summary
=============

Rackspace Auto Scale supports both schedule-based and event-based auto
scaling. Schedule-based auto scaling can be used to scale your
application up or down at a specific time of the day, whereas
event-based auto scaling can be used to automatically scale your
application up or down according to load. In this tutorial, we will use
Rackspace Orchestration to automate the configuration of both event-based and
schedule-based auto scaling.

Pre-reading
===========

If you are just getting started with Rackspace Auto Scale, Cloud
Monitoring, or Cloud Orchestration, the following introductory material
should give you enough background to proceed with this tutorial.

Auto Scale
----------

-  `Easily Scale Your Cloud With Rackspace Auto
   Scale <http://www.rackspace.com/blog/easily-scale-your-cloud-with-rackspace-auto-scale/>`__
   gives a brief overview of the different types of scaling policies for
   Auto Scale.
-  `Getting Started Guide: Auto Scale
   concepts <http://docs.rackspace.com/cas/api/v1.0/autoscale-gettingstarted/content/Auto_Scale_Core_Concepts.html>`__

Cloud Monitoring
----------------

-  `Getting Started Guide: How Rackspace Cloud Monitoring
   works <http://docs.rackspace.com/cm/api/v1.0/cm-getting-started/content/how-it-works-gsg.html>`__
-  `Getting Started Guide: About the monitoring
   agent <http://docs.rackspace.com/cm/api/v1.0/cm-getting-started/content/how-agent-works-gsg.html>`__
-  `Developer Guide: Available check types and
   fields <https://developer.rackspace.com/docs/cloud-monitoring/v1/developer-guide/#document-appendices/available-check-types-and-fields>`__
-  `Developer Guide: Server-Side agent configuration YAML file
   examples <https://developer.rackspace.com/docs/cloud-monitoring/v1/developer-guide/#document-appendices/server-side-agent-config-yaml>`__

Cloud Orchestration
-------------------

-  `Getting Started Guide:
   Overview <http://docs.rackspace.com/orchestration/api/v1/orchestration-getting-started/content/Orch_Overview.html>`__
-  `Templates Developer Guide: Introduction to
   templates <http://docs.rackspace.com/orchestration/api/v1/orchestration-templates-devguide/content/Intro_to_Templates-d1e633.html>`__

Schedule-based auto scaling
=================================

Schedule-based auto scaling can be useful if there is a particular time
when your application experiences a higher load. By scheduling a
scaling event, you will be able to proactively scale your application.

In this example, we will create a scaling group that scales up by one
node each Monday at 1:00 AM and down by one node each Saturday at 1:00
AM.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and Event-based scaling using Rackspace Cloud Autoscale

    resources:

    outputs:

Resources section
-----------------

Add an OS::Nova::KeyPair resource that will generate an SSH keypair
which you can use to login to your web servers if you need to:

.. code:: yaml

      access_key:
        type: OS::Nova::KeyPair
        properties:
          name: { get_param: "OS::stack_name" }
          save_private_key: true

Add a Rackspace::Cloud::LoadBalancer resource that will balance the load
between web servers.

.. code:: yaml

      scaling_lb:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: { get_param: "OS::stack_name" }
          protocol: HTTP
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4

Add a Rackspace::AutoScale::Group resource:

.. code:: yaml

      scaled_servers:
        type: Rackspace::AutoScale::Group
        properties:
          groupConfiguration:
            name: { get_param: "OS::stack_name" }
            maxEntities: 5
            minEntities: 1
            cooldown: 120
          launchConfiguration:
            type: launch_server
            args:
              loadBalancers:
              - loadBalancerId: { get_resource: scaling_lb }
                port: 80
              server:
                name: { get_param: "OS::stack_name" }
                flavorRef: performance1-1
                imageRef: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                key_name: { get_resource: access_key }
                networks:
                  - uuid: 11111111-1111-1111-1111-111111111111

This resource will be responsible for creating/destroying Cloud Servers
based on the auto scaling policy. The maxEntities and minEntities
properties above ensure that the group will create at least 1 server but
not more than 5 servers.

Add a Rackspace::AutoScale::ScalingPolicy for scaling up:

.. code:: yaml

      scale_up_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
          name:
            str_replace:
              template: stack scale up policy
              params:
                stack: { get_param: "OS::stack_name" }
          args:
            cron: "0 1 * * 1"
          change: 1
          cooldown: 600
          type: schedule

This resource will create a scaling policy that scales the auto scaling
group up by one server every Monday at 1:00 AM.

Finally, add a Rackspace::AutoScale::ScalingPolicy for scaling down:

.. code:: yaml

      scale_down_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
          name:
            str_replace:
              template: stack scale down policy
              params:
                stack: { get_param: "OS::stack_name" }
          args:
            cron: "0 1 * * 6"
          change: -1
          cooldown: 600
          type: schedule

Similarly, this resource will scale the auto scaling group down by one
server every Saturday at 1:00 AM.

Outputs section
---------------

Add the private SSH key to the outputs section. You will be able to log
into your scaling group servers using this SSH key.

.. code:: yaml

      "Access Private Key":
        value: { get_attr: [ access_key, private_key ] }
        description: Private key for accessing the scaled server instances if needed

To see the stack outputs, issue a ``heat stack-show <stack name>`` on
the created stack.

Full template
-------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and schedule-based scaling using Rackspace Cloud Autoscale

    resources:

      access_key:
        type: OS::Nova::KeyPair
        properties:
          name: { get_param: "OS::stack_name" }
          save_private_key: true

      scaling_lb:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: { get_param: "OS::stack_name" }
          protocol: HTTP
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4

      scaled_servers:
        type: Rackspace::AutoScale::Group
        properties:
          groupConfiguration:
            name: { get_param: "OS::stack_name" }
            maxEntities: 10
            minEntities: 2
            cooldown: 120
          launchConfiguration:
            type: launch_server
            args:
              loadBalancers:
              - loadBalancerId: { get_resource: scaling_lb }
                port: 80
              server:
                name: { get_param: "OS::stack_name" }
                flavorRef: performance1-1
                imageRef: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                key_name: { get_resource: access_key }
                networks:
                  - uuid: 11111111-1111-1111-1111-111111111111

      scale_up_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
          name:
            str_replace:
              template: stack scale up policy
              params:
                stack: { get_param: "OS::stack_name" }
          args:
            cron: "0 1 * * 1"
          change: 1
          cooldown: 600
          type: schedule

      scale_down_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
          name:
            str_replace:
              template: stack scale down policy
              params:
                stack: { get_param: "OS::stack_name" }
          args:
            cron: "0 1 * * 6"
          change: -1
          cooldown: 600
          type: schedule

    outputs:

      "Access Private Key":
        value: { get_attr: [ access_key, private_key ] }
        description: Private key for accessing the scaled server instances if needed

Event-based auto scaling
========================

To configure your web application running on the Rackspace Cloud to
automatically scale up or down according to load, Rackspace Auto Scale
can be used in conjunction with Rackspace Cloud Monitoring. The Cloud
Monitoring agent monitors various resources on the servers inside the
scaling group and makes calls to the Auto Scale API when it is time to
scale up or down.

In the following example template, we will set up a web application with
a load balancer and a scaling group that contains between 2 and 10 web
servers. For the sake of simplicity, we will not use template parameters
in this example.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and Event-based scaling using Rackspace Cloud Autoscale

    resources:

    outputs:

Resources section
-----------------

Add an OS::Nova::KeyPair resource and a Rackspace::Cloud::LoadBalancer
as in the previous example:

.. code:: yaml

      access_key:
        type: OS::Nova::KeyPair
        properties:
          name: { get_param: "OS::stack_name" }
          save_private_key: true

Add a Rackspace::Cloud::LoadBalancer resource that will balance the load
between web servers.

.. code:: yaml

      scaling_lb:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: { get_param: "OS::stack_name" }
          protocol: HTTP
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4

Autoscale resources
~~~~~~~~~~~~~~~~~~~

Add the Rackspace::AutoScale::Group resource, which will contain at least
2 servers and not more than 10 servers:

.. code:: yaml

      scaled_servers:
        type: Rackspace::AutoScale::Group
        properties:
          groupConfiguration:
            name: { get_param: "OS::stack_name" }
            maxEntities: 10
            minEntities: 2
            cooldown: 120
          launchConfiguration:
            type: launch_server
            args:
              loadBalancers:
              - loadBalancerId: { get_resource: scaling_lb }
                port: 80
              server:
                name: { get_param: "OS::stack_name" }
                flavorRef: performance1-1
                imageRef: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                key_name: { get_resource: access_key }
                config_drive: true
                networks:
                  - uuid: 11111111-1111-1111-1111-111111111111
                user_data:
                  str_replace:
                    template: |
                      #cloud-config
                      apt_upgrade: true
                      apt_sources:
                      - source: deb http://stable.packages.cloudmonitoring.rackspace.com/ubuntu-14.04-x86_64 cloudmonitoring main
                        key: |  # This is the apt repo signing key
                          -----BEGIN PGP PUBLIC KEY BLOCK-----
                          Version: GnuPG v1.4.10 (GNU/Linux)

                          mQENBFAZuVEBCAC8iXu/UEDLdkzRJzBKx14cgAiPHxSCjV4CPWqhOIrN4tl0PVHD
                          BYSJV7oSu0napBTfAK5/0+8zNnnq8j0PNg2YmPOFkL/rIMHJH8eZ08Ffq9j4GQdM
                          fSHDa6Zvgz68gJMLQ1IRPguen7p2mIEoOl8NuTwpjnWBZTdptImUoj53ZTKGYYS+
                          OWs2iZ1IHS8CbmWaTMxiEk8kT5plM3jvbkJAKBAaTfYsddo1JqqMpcbykOLcgSrG
                          oipyiDo9Ppi+EAOie1r6+zqmWpY+ScANkOpaVSfLjGp8fo4RP7gHhl26nDiqYB1K
                          7tV1Rl3RMPnGuh4g/8YRkiExKd/XdS2CfO/DABEBAAG0jFJhY2tzcGFjZSBDbG91
                          ZCBNb25pdG9yaW5nIEFnZW50IFBhY2thZ2UgUmVwbyAoaHR0cDovL3d3dy5yYWNr
                          c3BhY2UuY29tL2Nsb3VkL2Nsb3VkX2hvc3RpbmdfcHJvZHVjdHMvbW9uaXRvcmlu
                          Zy8pIDxtb25pdG9yaW5nQHJhY2tzcGFjZS5jb20+iQE4BBMBAgAiBQJQGblRAhsD
                          BgsJCAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRCghvB30Fq5FCo6B/9Oel0Q/cX6
                          1Lyk+teFywmB2jgn/UC51ioPZBHnHZLIjKH/CA6y7B9jm3+VddH60qDDANzlK/LL
                          MyUgwLj9+flKeS+H5AL6l3RarWlGm11fJjjW2TnaUCUXQxw6A/QQvpHpl7eknEKJ
                          m3kWMGAT6y/FbkSye18HUu6dtxvxosiMzi/7yVPJ7MwtUy2Bv1z9yHvt4I0rR8L5
                          CdFeEcqY4FlGmFBG200BuGzLMrqv6HF6LH3khPoXbGjVmHbHKIzqCx4hPWNRtZIv
                          fnu/aZcXJOJkB3/jzxaCjabOU+BCkXqVVFnUkbOYKoJ8EVLoepnhuVLUYErRjt7J
                          qDsI4KPQoEjTuQENBFAZuVEBCACUBBO83pdDYHfKe394Il8MSw7PBhtxFRHjUty2
                          WZYW12P+lZ3Q0Tqfc5Z8+CxnnkbdfvL13duAXn6goWObPRlQsYg4Ik9wO5TlYxqu
                          igtPZ+mJ9KlZZ/c2+KV4AeqO+K0L5k96nFkxd/Jh90SLk0ckP24RAYx2WqRrIPyX
                          xJCZlSWSqITMBcFp+kb0GdMk+Lnq7wPIJ08IKFJORSHgBbfHAmHCMOCUTZPhQHLA
                          yBDMLcaLP9xlRm72JG6tko2k2/cBV707CfbnR2PyJFqq+zuEyMdBpnxtY3Tpdfdk
                          MW9ScO40ndpwR72MG+Oy8iM8CTnmzRzMHMPiiPVAit1ZIXtZABEBAAGJAR8EGAEC
                          AAkFAlAZuVECGwwACgkQoIbwd9BauRSx0QgApV/n2L/Qe5T8aRhoiecs4gH+ubo2
                          uCQV9W3f56X3obHz9/mNkLTIKF2zHQhEUCCOwptoeyvmHht/QYXu1m3Gvq9X2F85
                          YU6I2PTEHuI/u6oZF7cEa8z8ofq91AWSOrXXEJiZUQr5DNjO8SiAzPulGM2teSA+
                          ez1wn9hhG9Kdu4LpaQ3EZHHBUKCLNU7nN/Ie5OeYA8FKbudNz13jTNRG+GYGrpPj
                          PlhA5RCmTY5N018O51YXEiTh4C7TLskFwRFPbbexh3mZx2s6VlcaCK0lEdQ/+XK3
                          KW+ZuPEh074b3VujLvuUCXd6T5FT5J6U/6qZgEoEiXwODX+fYIrD5PfjCw==
                          =S1lE
                          -----END PGP PUBLIC KEY BLOCK-----
                      write_files:
                      - path: /etc/rackspace-monitoring-agent.conf.d/load.yaml
                        content: |
                          type: agent.load_average
                          label: Load Average
                          period: 60
                          timeout: 10
                          alarms:
                            load_alarm:
                              label: load average alarm
                              notification_plan_id: {notification_plan}
                              criteria: |
                                :set consecutiveCount=3
                                if (metric['5m'] > 0.85){
                                    return new AlarmStatus(CRITICAL);
                                }
                                if (metric['15m'] < 0.3){
                                    return new AlarmStatus(WARNING);
                                }
                                return new AlarmStatus(OK);
                      - path: /etc/rackspace-monitoring-agent.cfg
                        content: |
                          monitoring_token {agent_token}
                      packages:
                      - rackspace-monitoring-agent
                      - apache2
                    params:
                      "{notification_plan}": { get_resource: scaling_plan }
                      "{agent_token}": { get_resource: agent_token }

In the resource above, the Cloud Monitoring agent is installed and
configured via the ``user_data`` section (using the `cloud-config
format <http://cloudinit.readthedocs.org/en/latest/topics/format.html#cloud-config-data>`__).
The alarm is configured to trigger a warning state when the system load
is below 0.3 for 15 minutes and a critical state when the system load is
above 0.85 for 5 minutes. We use the warning state here to trigger
scale-down events in lieu of an alternative alarm status.

The ``scaling_plan`` and ``agent_token`` resources referenced in the
``user_data`` section will be defined below.

Next, define a Rackspace::AutoScale::ScalingPolicy resource for scaling
up:

.. code:: yaml

      scale_up_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
          name:
            str_replace:
              template: stack scale up policy
              params:
                stack: { get_param: "OS::stack_name" }
          change: 1
          cooldown: 600
          type: webhook

Add a Rackspace::AutoScale::WebHook resource:

.. code:: yaml

      scale_up_webhook:
        type: Rackspace::AutoScale::WebHook
        properties:
          name:
            str_replace:
              template: stack scale up hook
              params:
                stack: { get_param: "OS::stack_name" }
          policy: { get_resource: scale_up_policy }

The webhook resource generates a URL that will be used to trigger the
scale-up policy above.

Similarly to the previous two resources for scaling-up, we will add
another Rackspace::AutoScale::ScalingPolicy and
Rackspace::AutoScale::WebHook resource for scaling down:

.. code:: yaml

      scale_down_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
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

Cloud Monitoring resources
~~~~~~~~~~~~~~~~~~~~~~~~~~

Add a Rackspace::CloudMonitoring::AgentToken resource that will create a
token used by the monitoring agent to authenticate with the monitoring
service:

.. code:: yaml

      agent_token:
        type: Rackspace::CloudMonitoring::AgentToken
        properties:
          label:
            str_replace:
              template: stack monitoring agent token
              params:
                stack: { get_param: "OS::stack_name" }

Add a Rackspace::CloudMonitoring::Notification resource that will call
the scale-up webhook created above:

.. code:: yaml

      scaleup_notification:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label:
            str_replace:
              template: stack scale up notification
              params:
                stack: { get_param: "OS::stack_name" }
          type: webhook
          details:
            url: { get_attr: [ scale_up_webhook, executeUrl ] }

Below, the notification resource will be associated with an alarm state
using a notification plan.

Add another Rackspace::CloudMonitoring::Notification resource that will
call the scale-down webhook:

.. code:: yaml

      scaledown_notification:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label:
            str_replace:
              template: stack scale down notification
              params:
                stack: { get_param: "OS::stack_name" }
          type: webhook
          details:
            url: { get_attr: [ scale_down_webhook, executeUrl ] }

Finally, create a Rackspace::CloudMonitoring::NotificationPlan and
Rackspace::CloudMonitoring::PlanNotifications resource.

.. code:: yaml

      scaling_plan:
        type: Rackspace::CloudMonitoring::NotificationPlan
        properties:
          label:
            str_replace:
              template: stack scaling notification plan
              params:
                stack: { get_param: "OS::stack_name" }

      plan_notifications:
        type: Rackspace::CloudMonitoring::PlanNotifications
        properties:
          plan: { get_resource: scaling_plan }
          warning_state: # scale down on warning since this is configured for low load
          - { get_resource: scaledown_notification }
          critical_state:
          - { get_resource: scaleup_notification }

The ``scaling_plan`` resource was referenced in the Cloud Monitoring
agent configuration inside of the ``user_data`` section of the
Rackspace::AutoScale::Group resource above. It tells the monitoring
agent how to respond to certain alarm states.

The Rackspace::CloudMonitoring::PlanNotifications resource is a way to
update an existing NotificationPlan resource. This allows us to
associate the alarm state with the Notification resource while avoiding
circular dependencies.

This notification plan will trigger a scale up event when any of the
``load_alarm`` alarms configured in the scaling group (via cloud-init)
issue a ``CRITICAL`` alarm state. This plan also triggers a scale down
event when any of the ``load_alarm`` alarms configured in the scaling
group issue a ``WARNING`` alarm state.

Outputs section
---------------

Add the private SSH key and, optionally, the webhook URLs to the outputs
section. You can use the webhooks to manually scale your scaling group
up or down.

.. code:: yaml

      "Access Private Key":
        value: { get_attr: [ access_key, private_key ] }
        description: Private key for accessing the scaled server instances if needed

      "Scale UP servers webhook":
        value: { get_attr: [ scale_up_webhook, executeUrl ] }
        description: Scale UP API servers webhook

      "Scale DOWN servers webhook":
        value: { get_attr: [ scale_down_webhook, executeUrl ] }

Full template
-------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and Event-based scaling using Rackspace Cloud Autoscale

    resources:

      access_key:
        type: OS::Nova::KeyPair
        properties:
          name: { get_param: "OS::stack_name" }
          save_private_key: true

      scaling_lb:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: { get_param: "OS::stack_name" }
          protocol: HTTP
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4

      scaled_servers:
        type: Rackspace::AutoScale::Group
        properties:
          groupConfiguration:
            name: { get_param: "OS::stack_name" }
            maxEntities: 10
            minEntities: 2
            cooldown: 120
          launchConfiguration:
            type: launch_server
            args:
              loadBalancers:
              - loadBalancerId: { get_resource: scaling_lb }
                port: 80
              server:
                name: { get_param: "OS::stack_name" }
                flavorRef: performance1-1
                imageRef: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                key_name: { get_resource: access_key }
                config_drive: true
                networks:
                  - uuid: 11111111-1111-1111-1111-111111111111
                user_data:
                  str_replace:
                    template: |
                      #cloud-config
                      apt_upgrade: true
                      apt_sources:
                      - source: deb http://stable.packages.cloudmonitoring.rackspace.com/ubuntu-14.04-x86_64 cloudmonitoring main
                        key: |  # This is the apt repo signing key
                          -----BEGIN PGP PUBLIC KEY BLOCK-----
                          Version: GnuPG v1.4.10 (GNU/Linux)

                          mQENBFAZuVEBCAC8iXu/UEDLdkzRJzBKx14cgAiPHxSCjV4CPWqhOIrN4tl0PVHD
                          BYSJV7oSu0napBTfAK5/0+8zNnnq8j0PNg2YmPOFkL/rIMHJH8eZ08Ffq9j4GQdM
                          fSHDa6Zvgz68gJMLQ1IRPguen7p2mIEoOl8NuTwpjnWBZTdptImUoj53ZTKGYYS+
                          OWs2iZ1IHS8CbmWaTMxiEk8kT5plM3jvbkJAKBAaTfYsddo1JqqMpcbykOLcgSrG
                          oipyiDo9Ppi+EAOie1r6+zqmWpY+ScANkOpaVSfLjGp8fo4RP7gHhl26nDiqYB1K
                          7tV1Rl3RMPnGuh4g/8YRkiExKd/XdS2CfO/DABEBAAG0jFJhY2tzcGFjZSBDbG91
                          ZCBNb25pdG9yaW5nIEFnZW50IFBhY2thZ2UgUmVwbyAoaHR0cDovL3d3dy5yYWNr
                          c3BhY2UuY29tL2Nsb3VkL2Nsb3VkX2hvc3RpbmdfcHJvZHVjdHMvbW9uaXRvcmlu
                          Zy8pIDxtb25pdG9yaW5nQHJhY2tzcGFjZS5jb20+iQE4BBMBAgAiBQJQGblRAhsD
                          BgsJCAcDAgYVCAIJCgsEFgIDAQIeAQIXgAAKCRCghvB30Fq5FCo6B/9Oel0Q/cX6
                          1Lyk+teFywmB2jgn/UC51ioPZBHnHZLIjKH/CA6y7B9jm3+VddH60qDDANzlK/LL
                          MyUgwLj9+flKeS+H5AL6l3RarWlGm11fJjjW2TnaUCUXQxw6A/QQvpHpl7eknEKJ
                          m3kWMGAT6y/FbkSye18HUu6dtxvxosiMzi/7yVPJ7MwtUy2Bv1z9yHvt4I0rR8L5
                          CdFeEcqY4FlGmFBG200BuGzLMrqv6HF6LH3khPoXbGjVmHbHKIzqCx4hPWNRtZIv
                          fnu/aZcXJOJkB3/jzxaCjabOU+BCkXqVVFnUkbOYKoJ8EVLoepnhuVLUYErRjt7J
                          qDsI4KPQoEjTuQENBFAZuVEBCACUBBO83pdDYHfKe394Il8MSw7PBhtxFRHjUty2
                          WZYW12P+lZ3Q0Tqfc5Z8+CxnnkbdfvL13duAXn6goWObPRlQsYg4Ik9wO5TlYxqu
                          igtPZ+mJ9KlZZ/c2+KV4AeqO+K0L5k96nFkxd/Jh90SLk0ckP24RAYx2WqRrIPyX
                          xJCZlSWSqITMBcFp+kb0GdMk+Lnq7wPIJ08IKFJORSHgBbfHAmHCMOCUTZPhQHLA
                          yBDMLcaLP9xlRm72JG6tko2k2/cBV707CfbnR2PyJFqq+zuEyMdBpnxtY3Tpdfdk
                          MW9ScO40ndpwR72MG+Oy8iM8CTnmzRzMHMPiiPVAit1ZIXtZABEBAAGJAR8EGAEC
                          AAkFAlAZuVECGwwACgkQoIbwd9BauRSx0QgApV/n2L/Qe5T8aRhoiecs4gH+ubo2
                          uCQV9W3f56X3obHz9/mNkLTIKF2zHQhEUCCOwptoeyvmHht/QYXu1m3Gvq9X2F85
                          YU6I2PTEHuI/u6oZF7cEa8z8ofq91AWSOrXXEJiZUQr5DNjO8SiAzPulGM2teSA+
                          ez1wn9hhG9Kdu4LpaQ3EZHHBUKCLNU7nN/Ie5OeYA8FKbudNz13jTNRG+GYGrpPj
                          PlhA5RCmTY5N018O51YXEiTh4C7TLskFwRFPbbexh3mZx2s6VlcaCK0lEdQ/+XK3
                          KW+ZuPEh074b3VujLvuUCXd6T5FT5J6U/6qZgEoEiXwODX+fYIrD5PfjCw==
                          =S1lE
                          -----END PGP PUBLIC KEY BLOCK-----
                      write_files:
                      - path: /etc/rackspace-monitoring-agent.conf.d/load.yaml
                        content: |
                          type: agent.load_average
                          label: Load Average
                          period: 60
                          timeout: 10
                          alarms:
                            load_alarm:
                              label: load average alarm
                              notification_plan_id: {notification_plan}
                              criteria: |
                                :set consecutiveCount=3
                                if (metric['5m'] > 0.85){
                                    return new AlarmStatus(CRITICAL);
                                }
                                if (metric['15m'] < 0.3){
                                    return new AlarmStatus(WARNING);
                                }
                                return new AlarmStatus(OK);
                      - path: /etc/rackspace-monitoring-agent.cfg
                        content: |
                          monitoring_token {agent_token}
                      packages:
                      - rackspace-monitoring-agent
                      - apache2
                    params:
                      "{notification_plan}": { get_resource: scaling_plan }
                      "{agent_token}": { get_resource: agent_token }

      scale_up_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
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
          group: { get_resource: scaled_servers }
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

      agent_token:
        type: Rackspace::CloudMonitoring::AgentToken
        properties:
          label:
            str_replace:
              template: stack monitoring agent token
              params:
                stack: { get_param: "OS::stack_name" }

      scaleup_notification:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label:
            str_replace:
              template: stack scale up notification
              params:
                stack: { get_param: "OS::stack_name" }
          type: webhook
          details:
            url: { get_attr: [ scale_up_webhook, executeUrl ] }

      scaledown_notification:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label:
            str_replace:
              template: stack scale down notification
              params:
                stack: { get_param: "OS::stack_name" }
          type: webhook
          details:
            url: { get_attr: [ scale_down_webhook, executeUrl ] }

      scaling_plan:
        type: Rackspace::CloudMonitoring::NotificationPlan
        properties:
          label:
            str_replace:
              template: stack scaling notification plan
              params:
                stack: { get_param: "OS::stack_name" }

      plan_notifications:
        type: Rackspace::CloudMonitoring::PlanNotifications
        properties:
          plan: { get_resource: scaling_plan }
          warning_state: # scale down on warning since this is configured for low load
          - { get_resource: scaledown_notification }
          critical_state:
          - { get_resource: scaleup_notification }


    outputs:

      "Access Private Key":
        value: { get_attr: [ access_key, private_key ] }
        description: Private key for accessing the scaled server instances if needed

      "Scale UP servers webhook":
        value: { get_attr: [ scale_up_webhook, executeUrl ] }
        description: Scale UP API servers webhook

      "Scale DOWN servers webhook":
        value: { get_attr: [ scale_down_webhook, executeUrl ] }

Auto-scaling using webhooks
===========================

If you decide to use a monitoring system other than Rackspace Cloud
Monitoring, you can remove the monitoring agent configuration from the
Rackspace::Autoscale::Group resource and remove the
Rackspace::CloudMonitoring resources. Be sure to include the webhooks in
the output values, as they will be needed when configuring monitoring.

Here is an example template for auto scaling with webhooks alone:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Rackspace Cloud Monitoring and Event-based scaling using Rackspace Cloud Autoscale

    resources:

      access_key:
        type: OS::Nova::KeyPair
        properties:
          name: { get_param: "OS::stack_name" }
          save_private_key: true

      scaling_lb:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: { get_param: "OS::stack_name" }
          protocol: HTTP
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4

      scaled_servers:
        type: Rackspace::AutoScale::Group
        properties:
          groupConfiguration:
            name: { get_param: "OS::stack_name" }
            maxEntities: 10
            minEntities: 2
            cooldown: 120
          launchConfiguration:
            type: launch_server
            args:
              loadBalancers:
              - loadBalancerId: { get_resource: scaling_lb }
                port: 80
              server:
                name: { get_param: "OS::stack_name" }
                flavorRef: performance1-1
                imageRef: 6f29d6a6-9972-4ae0-aa80-040fa2d6a9cf # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
                key_name: { get_resource: access_key }
                config_drive: true
                networks:
                  - uuid: 11111111-1111-1111-1111-111111111111

      scale_up_policy:
        type: Rackspace::AutoScale::ScalingPolicy
        properties:
          group: { get_resource: scaled_servers }
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
          group: { get_resource: scaled_servers }
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

    outputs:

      "Access Private Key":
        value: { get_attr: [ access_key, private_key ] }
        description: Private key for accessing the scaled server instances if needed

      "Scale UP servers webhook":
        value: { get_attr: [ scale_up_webhook, executeUrl ] }
        description: Scale UP API servers webhook

      "Scale DOWN servers webhook":
        value: { get_attr: [ scale_down_webhook, executeUrl ] }

Reference documentation
=======================

-  `Cloud Monitoring API Developer
   Guide <https://developer.rackspace.com/docs/cloud-monitoring/v1/developer-guide/>`__
-  `Auto Scale API Developer
   Guide <https://developer.rackspace.com/docs/autoscale/v1/developer-guide/>`__
-  `Cloud Orchestration API Developer
   Guide <https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide/>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud-init format
   documentation <http://cloudinit.readthedocs.org/en/latest/topics/format.html>`__

