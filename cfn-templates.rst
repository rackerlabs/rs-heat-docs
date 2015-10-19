===============================================
 Writing templates using CloudFormation format
===============================================

Overview
========

Rackspace Orchestration supports templates written in `AWS'
CloudFormation (CFN) format
<http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-guide.html>`_.
`CFN-compatible functions
<http://docs.openstack.org/developer/heat/template_guide/functions.html>`_
exist for most of the CFN functions (condition functions are not
supported).

In addition, the following four CFN resources are supported:

* `AWS::EC2::Instance <http://docs.openstack.org/developer/heat/template_guide/cfn.html#AWS::EC2::Instance>`_
* `AWS::ElasticLoadBalancing::LoadBalancer <http://docs.openstack.org/developer/heat/template_guide/cfn.html#AWS::ElasticLoadBalancing::LoadBalancer>`_
* `AWS::CloudFormation::WaitCondition <http://docs.openstack.org/developer/heat/template_guide/cfn.html#AWS::CloudFormation::WaitCondition>`_
* `AWS::CloudFormation::WaitConditionHandle <http://docs.openstack.org/developer/heat/template_guide/cfn.html#AWS::CloudFormation::WaitConditionHandle>`_

An AWS::EC2::Instance resource will result in a Rackspace Cloud Server
being created, and a AWS::ElasticLoadBalancing::LoadBalancer resource
will result in a Rackspace Cloud Loadbalancer being created.  The wait
condition resources are used internally as a signaling mechanism and
do not map to any cloud resources.


Writing a CFN template
======================

CFN templates are written in JSON format.  Here is an example of a
template that creates a server and executes a bash script on it:

.. code:: json

    {
      "AWSTemplateFormatVersion" : "2010-09-09",
      "Description" : "Hello world",
      "Parameters" : {
        "InstanceType" : {
          "Description" : "WebServer EC2 instance type",
          "Type" : "String",
          "Default" : "1GB Standard Instance",
          "AllowedValues" : [ "1GB Standard Instance", "2GB Standard Instance" ],
          "ConstraintDescription" : "must be a valid EC2 instance type."
        }
      },
      "Resources" : {
        "TestServer": {
          "Type": "AWS::EC2::Instance",
          "Properties": {
            "ImageId" : "4b14a92e-84c8-4770-9245-91ecb8501cc2",
            "InstanceType"   : { "Ref" : "InstanceType" },
            "UserData"       : { "Fn::Base64" : { "Fn::Join" : ["", [
              "#!/bin/bash -v\n",
              "echo \"hello world\" > /root/hello-world.txt\n"
            ]]}}
          }
        }
      },
      "Outputs" : {
        "PublicIP" : {
          "Value" : { "Fn::GetAtt" : [ "TestServer", "PublicIp" ]},
          "Description" : "Public IP of server"
        }
      }
    }

Notice that the "InstanceType" must be a valid Cloud Server flavor
("m1.small", for example, is not).  Also, the "ImageId" property must
be a valid Cloud Server image ID or image name.

Using the CFN resources
=======================

It's possible to use a CFN resource in a HOT template.  In this
example, we will create an AWS::EC2::Instance, keep track of the
user_data script's progress using AWS::CloudFormation::WaitCondition
and AWS::CloudFormation::WaitConditionHandle, and add the server to
an AWS::ElasticLoadBalancing::LoadBalancer.

.. code:: yaml

    heat_template_version: 2014-10-16 
    
    description: |
      Test template for AWS supported resources 
    
    resources:
    
      aws_server1:
        type: AWS::EC2::Instance
        properties:
          ImageId: 753a7703-4960-488b-aab4-a3cdd4b276dc # Ubuntu 14.04 LTS (Trusty Tahr) (PVHVM)
          InstanceType: 4 GB Performance 
          UserData:
            str_replace:
              template: |
                #!/bin/bash
                apt-get update
                apt-get -y install curl
                sleep 2
                curl -i -X PUT --data-binary '{"status": "SUCCESS", "reason": "AWS Signal"}' "wc_notify"
              params:
                wc_notify: { get_resource: aws_handle }
      
      aws_handle:
        type: AWS::CloudFormation::WaitConditionHandle
      
      aws_wait_condition:
        type: AWS::CloudFormation::WaitCondition
        properties:
          Handle: { get_resource: aws_handle }
          Timeout: 600
    
      elastic_load_balancer:
            type: AWS::ElasticLoadBalancing::LoadBalancer
            properties:
                AvailabilityZones: []
                Instances: [ get_resource: aws_server1 ]
                Listeners: [{
                    LoadBalancerPort: 8945,
                    InstancePort: 80,
                    Protocol: "HTTP"
                }]
                HealthCheck:
                    Target: "HTTP:80/"
                    HealthyThreshold: 3
                    UnhealthyThreshold: 10
                    Interval: 10
                    Timeout: 60
    
    outputs:
    
      "AWS Server ID":
        value: { get_resource: aws_server1 }
        description: ID of the AWS::EC2::Instance resource
    
      "AWS EC2 Server AvailabilityZone":
        value: { get_attr: [ aws_server1, AvailabilityZone ] }
        description: AWS EC2 Server AvailabilityZone 
    
      "AWS EC2 Server PrivateDnsName":
        value: { get_attr: [ aws_server1, PrivateDnsName ] }
        description: AWS EC2 Server PrivateDnsName 
    
      "AWS EC2 Server PrivateIp":
        value: { get_attr: [ aws_server1, PrivateIp ] }
        description: AWS EC2 Server PrivateIp 
    
      "AWS EC2 Server PublicDnsName":
        value: { get_attr: [ aws_server1, PublicDnsName ] }
        description: AWS EC2 Server PublicDnsName 
    
      "AWS EC2 Server PublicIp":
        value: { get_attr: [ aws_server1, PublicIp ] }
        description: AWS EC2 Server PublicIp 
    
      "AWS Cloud Formation Wait Condition":
        value: { get_attr: [ aws_wait_condition, Data ] }
        description: AWS Cloud Formation Wait Condition data 
    
      "AWS ElasticLoadBalancer CanonicalHostedZoneName":
        value: { get_attr: [ elastic_load_balancer, CanonicalHostedZoneName ] }
        description: details the CanonicalHostedZoneName 
    
      "AWS ElasticLoadBalancer CanonicalHostedZoneNameID":
        value: { get_attr: [ elastic_load_balancer, CanonicalHostedZoneNameID ] }
        description: details the CanonicalHostedZoneNameID 
    
      "AWS ElasticLoadBalancer DNSName":
        value: { get_attr: [ elastic_load_balancer, DNSName ] }
        description: details the DNSName 

Likewise, you can use HOT resources in a CFN template.  In this
example, an OS::Nova::Server resource is embedded in a CFN template.

.. code:: json

    {
      "AWSTemplateFormatVersion" : "2010-09-09",
      "Description" : "Hello world",
      "Parameters" : {
        "InstanceType" : {
          "Description" : "WebServer EC2 instance type",
          "Type" : "String",
          "Default" : "1GB Standard Instance",
          "AllowedValues" : [ "1GB Standard Instance", "2GB Standard Instance" ],
          "ConstraintDescription" : "must be a valid EC2 instance type."
        }
      },
      "Resources" : {
        "TestServer": {
          "Type": "OS::Nova::Server",
          "Properties": {
            "image" : "4b14a92e-84c8-4770-9245-91ecb8501cc2",
            "flavor" : { "Ref" : "InstanceType" },
            "config_drive" : "true",
            "user_data_format" : "RAW",
            "user_data" : { "Fn::Base64" : { "Fn::Join" : ["", [
              "#!/bin/bash -v\n",
              "echo \"hello world\" > /root/hello-world.txt\n"
            ]]}}
          }
        }
      },
      "Outputs" : {
        "PublicIP" : {
          "Value" : { "Fn::GetAtt" : [ "TestServer", "accessIPv4" ]},
          "Description" : "Public IP of server"
        }
      }
    }
