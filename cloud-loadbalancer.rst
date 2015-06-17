==============================
 Rackspace Cloud Loadbalancer
==============================

Note: This document assumes that the reader is familiar with HOT
specification. If that is not the case, please go to 'References'
section given at the end of this document for HOT specification link.

Brief summary
=============

A loadbalancer is used to distribute workloads between multiple back-end
systems or services based on the criteria defined as part of its's
configuration.

Example loadbalancer template
=============================

Simple loadbalancer template is given below.

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Creating Rackspace cloud loadbalancer using orchestration service.

    resources:
      api_loadbalancer:
        type: Rackspace::Cloud::LoadBalancer
        properties:
          name: test_load_balancer
          metadata:
            rax-heat: { get_param: "OS::stack_id" }
          protocol: HTTPS
          port: 80
          algorithm: ROUND_ROBIN
          nodes: []
          virtualIps:
          - type: SERVICENET
            ipVersion: IPV4

Loadbalancer properties
=======================

Below given is the complete list of loadbalancer properties that can be
provided to the resource.

.. code:: yaml

      accessList: {Type: CommaDelimitedList}
      algorithm:
        AllowedValues: [LEAST_CONNECTIONS, RANDOM, ROUND_ROBIN, WEIGHTED_LEAST_CONNECTIONS,
          WEIGHTED_ROUND_ROBIN]
        Type: String
      connectionLogging:
        AllowedValues: ['True', 'true', 'False', 'false']
        Type: Boolean
      connectionThrottle: {Type: Json}
      contentCaching:
        AllowedValues: [ENABLED, DISABLED]
        Type: String
      errorPage: {Type: String}
      halfClosed:
        AllowedValues: ['True', 'true', 'False', 'false']
        Type: Boolean
      healthMonitor: {Type: Json}
      httpsRedirect:
        AllowedValues: ['True', 'true', 'False', 'false']
        Default: false
        Description: Enables or disables HTTP to HTTPS redirection for the load balancer.
          When enabled, any HTTP request returns status code 301 (Moved Permanently),
          and the requester is redirected to the requested URL via the HTTPS protocol
          on port 443. Only available for HTTPS protocol (port=443), or HTTP protocol
          with a properly configured SSL termination (secureTrafficOnly=true, securePort=443).
        Type: Boolean
      metadata: {Type: Json}
      name: {Type: String}
      nodes: 
        Type: CommaDelimitedList
        Required: True
      port: 
        Type: Number
        Required: True
      protocol:
        AllowedValues: [DNS_TCP, DNS_UDP, FTP, HTTP, HTTPS, IMAPS, IMAPv4, LDAP, LDAPS,
          MYSQL, POP3, POP3S, SMTP, TCP, TCP_CLIENT_FIRST, UDP, UDP_STREAM, SFTP]
        Type: String
        Required: True
      sessionPersistence:
        AllowedValues: [HTTP_COOKIE, SOURCE_IP]
        Type: String
      sslTermination: {Type: Json}
      timeout: {MaxValue: 120, MinValue: 1, Type: Number}
      virtualIps: 
        MinLength: 1
        Type: CommaDelimitedList
        Required: True

Example template with loadbalancer
==================================

In the following example template, we will create a multi node wordpress
application with two Linux servers, one Trove (DBaaS) instance and one
Loadbalancer.

First add a database instance resource to the template.

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Creating Rackspace cloud server with user_data.

    resources:
      db:
        type: OS::Trove::Instance
        properties:
          name: wordpress
          flavor: 1GB Instance
          size: 30
          users:
          - name: admin
            password: admin
            databases:
            - wordpress
          databases:
          - name: wordpress      

This template creates a database instance with name 'wordpress' and
'admin' as the username and password.

Now add two server resoruces and install wordpress application.

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Creating Rackspace cloud server with SSH access private key.

    resources:
      web_nodes:
        type: OS::Heat::ResourceGroup
        properties:
          count: 2
          resource_def:
            type: "OS::Nova::Server"
            properties:
              name: test-server
              flavor: 2 GB General Purpose v1
              image: Debian 7 (Wheezy) (PVHVM)
              user_data:
                str_replace:
                  template: |
                    #!/bin/bash -v
                    yum -y install mysql-server httpd wordpress
                    sed -i "/Deny from All/d" /etc/httpd/conf.d/wordpress.conf
                    sed -i "s/Require local/Require all granted/" /etc/httpd/conf.d/wordpress.conf
                    sed --in-place --e "s/localhost/%dbhost%/" --e "s/database_name_here/%dbname%/" --e "s/username_here/%dbuser%/" --e "s/password_here/%dbpass%/" /usr/share/wordpress/wp-config.php
                    /etc/init.d/httpd start
                    chkconfig httpd on
                    /etc/init.d/mysqld start
                    chkconfig mysqld on
                    cat << EOF | mysql
                    CREATE DATABASE %dbname%;
                    GRANT ALL PRIVILEGES ON %dbname%.* TO "%dbuser%"@"localhost"
                    IDENTIFIED BY "%dbpass%";
                    FLUSH PRIVILEGES;
                    EXIT
                    EOF
                    iptables -I INPUT -p tcp --dport 80 -j ACCEPT
                    iptables-save > /etc/sysconfig/iptables
                  params:
                    "%dbhost%": { get_attr: [ db, hostname ] }
                    "%dbname%": wordpress
                    "%dbuser%": admin
                    "%dbpass%": admin
      db:
        type: OS::Trove::Instance
        properties:
          name: wordpress
          flavor: 1GB Instance
          size: 30
          users:
          - name: admin
            password: admin
            databases:
            - wordpress
          databases:
          - name: wordpress  

Here a ResourceGroup of type 'OS::Nova::Server' is added to the
template. The user_data property contains a script to install the
wordpress application. Please note that database instance hostname
information is passed to the script.

Finally, add the loadbalancer resource and provide the server addresses
to the loadbalancer. Given below is the complete template that can be
used to create a loadbalanced multi node wordpress application.

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Create a loadbalanced two node wordpress application.

    resources:
      lb:
        type: "Rackspace::Cloud::LoadBalancer"
        properties:
          name: wordpress_loadbalancer
          nodes:
          - addresses: { get_attr: [ web_nodes, privateIPv4 ] }
            port: 80
            condition: ENABLED
          protocol: HTTP
          halfClosed: False
          algorithm: LEAST_CONNECTIONS
          connectionThrottle:
            maxConnections: 50
            minConnections: 50
            maxConnectionRate: 50
            rateInterval: 50
          port: 80
          timeout: 120
          sessionPersistence: HTTP_COOKIE
          virtualIps:
          - type: PUBLIC
            ipVersion: IPV4
          healthMonitor:
            type: HTTP
            delay: 10
            timeout: 10
            attemptsBeforeDeactivation: 3
            path: "/"
            statusRegex: "."
            bodyRegex: "."
          contentCaching: ENABLED

      web_nodes:
        type: OS::Heat::ResourceGroup
        properties:
          count: 2
          resource_def:
            type: "OS::Nova::Server"
            properties:
              name: test-server
              flavor: 2 GB General Purpose v1
              image: Debian 7 (Wheezy) (PVHVM)
              user_data:
                str_replace:
                  template: |
                    #!/bin/bash -v
                    yum -y install mysql-server httpd wordpress
                    sed -i "/Deny from All/d" /etc/httpd/conf.d/wordpress.conf
                    sed -i "s/Require local/Require all granted/" /etc/httpd/conf.d/wordpress.conf
                    sed --in-place --e "s/localhost/%dbhost%/" --e "s/database_name_here/%dbname%/" --e "s/username_here/%dbuser%/" --e "s/password_here/%dbpass%/" /usr/share/wordpress/wp-config.php
                    /etc/init.d/httpd start
                    chkconfig httpd on
                    /etc/init.d/mysqld start
                    chkconfig mysqld on
                    cat << EOF | mysql
                    CREATE DATABASE %dbname%;
                    GRANT ALL PRIVILEGES ON %dbname%.* TO "%dbuser%"@"localhost"
                    IDENTIFIED BY "%dbpass%";
                    FLUSH PRIVILEGES;
                    EXIT
                    EOF
                    iptables -I INPUT -p tcp --dport 80 -j ACCEPT
                    iptables-save > /etc/sysconfig/iptables
                  params:
                    "%dbhost%": { get_attr: [ db, hostname ] }
                    "%dbname%": wordpress
                    "%dbuser%": admin
                    "%dbpass%": admin
      db:
        type: OS::Trove::Instance
        properties:
          name: wordpress
          flavor: 1GB Instance
          size: 30
          users:
          - name: admin
            password: admin
            databases:
            - wordpress
          databases:
          - name: wordpress

    outputs:
      wordpress_url:
        value: 
          str_replace:
            template: "http://%ip%/wordpress"
            params:
              "%ip%": { get_attr: [ lb, PublicIp ] }
        description: Public URL for the wordpress blog      

Please note that, to keep the template simple, all the values were hard
coded in the above template.

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/overview.html>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud load balancer getting started
   guide <http://docs.rackspace.com/loadbalancers/api/v1.0/clb-getting-started/content/LB_Overview.html>`__
-  `Cloud load balancer API developer
   guide <http://docs.rackspace.com/loadbalancers/api/v1.0/clb-devguide/content/Overview-d1e82.html>`__

