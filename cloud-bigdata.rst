=========================
 Rackspace Cloud Big Data
=========================

Overview
========

This project defines the `Rackspace Cloud Big Data <http://www.rackspace.com/en-us/cloud/big-data>`__ (CBD) team's `Cloud Orchestration <http://www.rackspace.com/en-us/cloud/orchestration>`__ (`OpenStack Heat <https://wiki.openstack.org/wiki/Heat>`__) Resource Plugin. The CBD plugin enables accessing Rackspace Cloud Big Data services through `Heat Orchestration Templates <http://docs.openstack.org/developer/heat/template_guide/hot_guide.html>`__ (HOTs) by editing a YAML file or through the `Rackspace Control Panel <https://mycloud.rackspace.com/>`__ Orchestration management interface. The Rackspace Cloud Big Data Heat resource plugin identifier is:

**Rackspace::Cloud::BigData**

The CBD plugin may be used for creating and deleting CBD services.

Example Template
================

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2015-10-15

    description: |
      This is a Heat template to deploy a Rackspace::Cloud::BigData cluster.

    resources:

    outputs:

Resources Section
-----------------
This section defines Rackspace::Cloud::BigData required property configuration values. For a full description of each property field, please see the Cloud Big Data Developer Guide linked at the bottom of this page but this is a quick overview:

- **stackId**: Stack IDs define the type of CBD cluster to create. The example below creates an `Apache Hadoop <https://hadoop.apache.org/>`__-based `Hortonworks Data Platform <http://hortonworks.com/hdp/>`__ cluster (HADOOP_HDP2_2). Stack IDs are being added fairly quickly so it is always best to query the CBD REST API for the current list of supported Rackspace stacks (see Reference Documentation below for details).
- **clusterName**: A descriptive name of the cluster; useful when there are multiple clusters to track
- **clusterLogin**: This will be the account created on the cluster to enable SSH access. Example: ssh clusterLogin@<your cluster IP>.
- **flavor**: The flavor defines the resources (like disk space and RAM) available on the individual systems making up the cluster. Flavors are updated when new cloud hardware is available so it is always best to check the CBD REST API for the current list (see Reference Documentation below for details).
- **numSlaveNodes**: numSlaveNodes defines the replication factor (how many copies of your data are stored for fault tolerance). 3 tends to be a pretty standard value. Higher replication will mean less total disk space available in the cluster and lower values may increase risk of data loss in rare catastrophic failure conditions.
- **publicKeyName**: publicKeyName is an easy to remember name for the SSH public key used in the cluster. This name may be reused in other clusters in order to reuse public keys and allow the same private key to SSH into clusters across a company/organization if desired.
- **publicKey**: This is the full public key (matching your securely stored private key) that will be installed into the cluster systems which enable SSHing into cluster nodes. CBD clusters disable login/password-based SSH connections so this key is required for cluster access. It is important to put quotes around this value.

The following YAML shows what a filled in example resource section might look like:

.. code:: yaml

      cbd_cluster:
        type: "Rackspace::Cloud::BigData"
        properties:
          stackId:       HADOOP_HDP2_2
          clusterName:   my_cbd_cluster
          clusterLogin:  my_ssh_login
          flavor:        Small Hadoop Instance
          numSlaveNodes: 3
          publicKeyName: my_ssh_pub_key_name
          publicKey:     "... paste public SSH key here ..."


Outputs Section
---------------

The output fields provide information about the cluster creation process. cbd_version is the version of the CBD system used to generate the cluster. This is just an informative field. cluster_id is the important value to retain about the cluster. It is the unique identifier of the cluster within the CBD system and is used in CBD REST API calls (for more information on the CBD REST API, see the Cloud Big Data Developer Guide at the bottom of this document). cluster_id is usually in a `UUID <https://en.wikipedia.org/wiki/Universally_unique_identifier>`__ format.

Outputs section example:

.. code:: yaml

      cluster_id:
        value: { get_resource: cbd_cluster }
        description: Cloud Big Data Cluster ID

      cbd_version:
        value: { get_attr: [cbd_cluster, cbdVersion] }
        description: Cloud Big Data version


Full Template
-------------

.. code:: yaml

    heat_template_version: 2015-10-15

    description: |
      This is a Heat template to deploy a Rackspace::Cloud::BigData cluster.

    resources:
      cbd_cluster:
        type: "Rackspace::Cloud::BigData"
        properties:
          stackId:       HADOOP_HDP2_2
          clusterName:   my_cbd_cluster
          clusterLogin:  my_ssh_login
          flavor:        Small Hadoop Instance
          numSlaveNodes: 3
          publicKeyName: my_ssh_pub_key_name
          publicKey:     "... paste public SSH key here ..."

    outputs:
      cluster_id:
        value: { get_resource: cbd_cluster }
        description: Cloud Big Data Cluster ID

      cbd_version:
        value: { get_attr: [cbd_cluster, cbdVersion] }
        description: Cloud Big Data version

Reference Documentation
=======================

- `Rackspace Cloud Big Data Developer Guide, v2 <http://docs.rackspace.com/cbd/api/v1.0/cbd-devguide-2/cbd-devguide-2-20150630.pdf>`__
- `Example CBD Flavor Names <http://docs.rackspace.com/cbd/api/v1.0/cbd-getting-started-2/content/client_listFlavor.html>`__
   - The "Name" column shows example flavor names but please always consult the CBD API for up to date flavor information
- `Example Cloud Big Data Stack IDs <http://docs.cloudbigdataplatform.com/v1_v2.html#supported-operations>`__
   - The "Cluster Type" column provides example Stack IDs but please always consult the CBD API for up to date information
- `Cloud Big Data Python CLI <https://github.com/rackerlabs/python-lavaclient>`__
   - A useful tool for accessing CBD data such as the most current flavor and stack IDs
- `Cloud Big Data Heat Resource Plugin Github <https://github.com/rackerlabs/cbd_heat_plugin>`__
