=====================================
 Cloud Monitoring resources for Heat
=====================================

Brief summary
=============

The Rackspace Cloud Monitoring resources allow you to configure
monitoring on resources that you create with Heat.

Pre-reading
===========

The following introductory material should give you enough background to
proceed with this tutorial.

-  `Rackspace Cloud Monitoring Getting Started
   Guide <http://docs.rackspace.com/cm/api/v1.0/cm-getting-started/cm-getting-started-20150316.pdf>`__
   (especially "How Rackspace Cloud Monitoring works" on page 8)
-  `Getting Started With Rackspace Monitoring
   CLI <http://www.rackspace.com/knowledge_center/article/getting-started-with-rackspace-monitoring-cli>`__
-  `Rackspace Cloud Monitoring Checks and
   Alarms <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-monitoring-checks-and-alarms>`__

-  Example template

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2013-05-23

    description: |
      Test template using Cloud Monitoring

    resources:

Resources section
-----------------

Add an OS::Nova::Server resource and configure it to install the Cloud
Monitoring agent:

.. code:: yaml

      server:
        type: OS::Nova::Server
        properties:
          image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
          flavor: 2 GB Performance
          name: { get_param: "OS::stack_name" }
          user_data_format: RAW
          config_drive: true
          user_data:
            str_replace:
              template: |
                #!/bin/bash
                echo "deb http://stable.packages.cloudmonitoring.rackspace.com/ubuntu-14.04-x86_64 cloudmonitoring main" > /etc/apt/sources.list.d/rackspace-monitoring-agent.list
                curl https://monitoring.api.rackspacecloud.com/pki/agent/linux.asc | sudo apt-key add -
                apt-get -y update
                apt-get -y install rackspace-monitoring-agent apache2
                echo "monitoring_token {{token}}" > /etc/rackspace-monitoring-agent.cfg
                service rackspace-monitoring-agent restart
              params:
                "{{token}}": { get_resource: token }
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }

It is possible to monitor one or more servers by creating a
Rackspace::CloudMonitoring::Entity resource [1]_. Entities are
automatically created for Cloud Servers, so we will refer to the server
resource above as our Cloud Monitoring entity.

Add a Rackspace::CloudMonitoring::AgentToken resource that will create a
token used by the monitoring agent to authenticate with the monitoring
service:

.. code:: yaml

      token:
        type: Rackspace::CloudMonitoring::AgentToken
        properties:
          label: { get_param: "OS::stack_name" }

Add a Rackspace::CloudMonitoring::Check resource and configure it to
check that the web service on the server entity is responsive:

.. code:: yaml

      webcheck:
        type: Rackspace::CloudMonitoring::Check
        properties:
          entity: { get_resource: server }
          type: remote.http
          details:
            url:
              str_replace:
                template: http://server_ip/
                params:
                  server_ip: { get_attr: [ server, accessIPv4 ] }
          label: webcheck
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }
          period: 120
          timeout: 10
          monitoring_zones_poll:
          - Northern Virginia (IAD)
          - Chicago (ORD)
          target_hostname: { get_attr: [ server, accessIPv4 ] }
          target_receiver: IPv4

Add another Rackspace::CloudMonitoring::Check resource and configure it
to check the server's CPU resources via the monitoring agent:

.. code:: yaml

      cpucheck:
        type: Rackspace::CloudMonitoring::Check
        properties:
          entity: { get_resource: server }
          type: agent.cpu
          label: cpu_check
          details: {}
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }
          period: 30
          timeout: 10

The actual alarm criteria for the CPU check will be defined in the
Rackspace::CloudMonitoring::Alarm resource below.

Add a Rackspace::CloudMonitoring::Notification resource that will send
an email to admin@example.com whenever it is triggered:

.. code:: yaml

      email_notification_1:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label: email_ops_team
          type: email
          details:
            address: "admin@example.com"

Add a similar Rackspace::CloudMonitoring::Notification resource that
will send an email to allclear@example.com whenever it is triggered:

.. code:: yaml

      email_notification_2:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label: email_ops_team_2
          type: email
          details:
            address: "allclear@example.com"

Add a Rackspace::CloudMonitoring::NotificationPlan resource to
configure Cloud Monitoring to trigger the email_notification1
notification whenever an alarm enters the WARNING or CRITICAL state
and email_notification2 whenever an alarm enters the OK state:

.. code:: yaml

      notify_ops_team:
        type: Rackspace::CloudMonitoring::NotificationPlan
        properties:
          label: { get_param: "OS::stack_name" }
          warning_state:
          - { get_resource: email_notification_1 }
          critical_state:
          - { get_resource: email_notification_1 }
          ok_state:
          - { get_resource: email_notification_2 }

Finally, add a Rackspace::CloudMonitoring::Alarm resource that will
configure the agent to enter the WARNING state when CPU usage is over
85% for 5 seconds, the CRITICAL state when CPU usage is over 95% for 5
seconds, and the OK state otherwise:

.. code:: yaml

      alert_ops:
        type: Rackspace::CloudMonitoring::Alarm
        properties:
          label: test_cpu_alarm
          check: { get_resource: cpucheck }
          plan: { get_resource: notify_ops_team }
          criteria: |
            :set consecutiveCount=5
            if (metric['usage_average'] > 95) {
                return new AlarmStatus(CRITICAL, 'CPU usage is #{usage_average}%');
            }
            if (metric['usage_average'] > 85) {
                return new AlarmStatus(WARNING, 'CPU usage is #{usage_average}%');
            }
            return new AlarmStatus(OK);
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }

Full template
-------------

.. code:: yaml

    heat_template_version: 2013-05-23

    description: |
      Test template using Cloud Monitoring

    resources:

      server:
        type: OS::Nova::Server
        properties:
          image: Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
          flavor: 2 GB Performance
          name: { get_param: "OS::stack_name" }
          user_data_format: RAW
          config_drive: true
          user_data:
            str_replace:
              template: |
                #!/bin/bash
                echo "deb http://stable.packages.cloudmonitoring.rackspace.com/ubuntu-14.04-x86_64 cloudmonitoring main" > /etc/apt/sources.list.d/rackspace-monitoring-agent.list
                curl https://monitoring.api.rackspacecloud.com/pki/agent/linux.asc | sudo apt-key add -
                apt-get -y update
                apt-get -y install rackspace-monitoring-agent apache2
                echo "monitoring_token {{token}}" > /etc/rackspace-monitoring-agent.cfg
                service rackspace-monitoring-agent restart
              params:
                "{{token}}": { get_resource: token }
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }

      token:
        type: Rackspace::CloudMonitoring::AgentToken
        properties:
          label: { get_param: "OS::stack_name" }

      webcheck:
        type: Rackspace::CloudMonitoring::Check
        properties:
          entity: { get_resource: server }
          type: remote.http
          details:
            url:
              str_replace:
                template: http://server_ip/
                params:
                  server_ip: { get_attr: [ server, accessIPv4 ] }
          label: webcheck
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }
          period: 120
          timeout: 10
          monitoring_zones_poll:
          - Northern Virginia (IAD)
          - Chicago (ORD)
          target_hostname: { get_attr: [ server, accessIPv4 ] }
          target_receiver: IPv4

      cpucheck:
        type: Rackspace::CloudMonitoring::Check
        properties:
          entity: { get_resource: server }
          type: agent.cpu
          label: cpu_check
          details: {}
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }
          period: 30
          timeout: 10

      email_notification_1:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label: email_ops_team
          type: email
          details:
            address: "admin@example.com"

      email_notification_2:
        type: Rackspace::CloudMonitoring::Notification
        properties:
          label: email_ops_team_2
          type: email
          details:
            address: "allclear@example.com"

      notify_ops_team:
        type: Rackspace::CloudMonitoring::NotificationPlan
        properties:
          label: { get_param: "OS::stack_name" }
          warning_state:
          - { get_resource: email_notification_1 }
          critical_state:
          - { get_resource: email_notification_1 }
          ok_state:
          - { get_resource: email_notification_2 }

      alert_ops:
        type: Rackspace::CloudMonitoring::Alarm
        properties:
          label: test_cpu_alarm
          check: { get_resource: cpucheck }
          plan: { get_resource: notify_ops_team }
          criteria: |
            :set consecutiveCount=5
            if (metric['usage_average'] > 95) {
                return new AlarmStatus(CRITICAL, 'CPU usage is #{usage_average}%');
            }
            if (metric['usage_average'] > 85) {
                return new AlarmStatus(WARNING, 'CPU usage is #{usage_average}%');
            }
            return new AlarmStatus(OK);
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
            stack-name: { get_param: "OS::stack_name" }

Reference documentation
=======================

-  `Rackspace Cloud Monitoring Developer
   Guide <http://docs.rackspace.com/cm/api/v1.0/cm-devguide/content/index.html>`__

.. [1]
   The following is an example of a Rackspace::CloudMonitoring::Entity
   resource definition:

   .. code:: yaml

         entity:
           type: Rackspace::CloudMonitoring::Entity
           properties:
             label: { get_param: "OS::stack_name" }
             metadata:
               rax-heat: { get_param: "OS::stack_id" }
               stack-name: { get_param: "OS::stack_name" }
             ip_addresses:
               web_server: { get_attr: [ server, accessIPv4 ] }

