===================================
 Rackspace Cloud Backup
===================================

Brief summary
=============

Rackspace cloud backup configuration resource enables you to select and
backup specific files/folders from a cloud server using Cloud
Orchestration.

Prerequisite(s):
================

Cloud backup agent is installed on the server from where you want to
backup files/folders.

Installing cloud backup agent on the server
-------------------------------------------

Option-1
~~~~~~~~

If the server from which you want to create a backup was created as part of
a Heat stack, then pass '{build_config: backup_agentonly}' as metadata
to OS::Nova::Server or Rackspace::Cloud::WinServer. For example,

.. code:: yaml

      wordpress_server:
        type: "Rackspace::Cloud::WinServer"
        properties:
          name: wordpress-server
          flavor: 4GB Standard Instance
          image: Windows Server 2012
          metadata: {build_config: backup_agent_only}

Option-2
~~~~~~~~

If the server was not created as part of a Heat stack, then follow
the links given below to install backup agent manually.

-  `Install cloud backup agent on Linux
   server <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-linux>`__
-  `Install cloud backup agent on Windows
   server <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-windows>`__
-  `Install cloud backup agent on Windows server (silent
   installation) <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-windows-by-using-silent-installation>`__

Example template
================

In the following example template, we will set up a single node
WordPress web application (on a Windows server) with a cloud backup
resource. For the sake of simplicity, we will not use template
parameters in this example.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Wordpress application on a Windows server with cloud backup enabled.

    resources:

    outputs:

Resources section
-----------------

Add a Rackspace::Cloud::WinServer resource that will create a Windows
server and install the WordPress web application.

.. code:: yaml

      wordpress_server:
        type: "Rackspace::Cloud::WinServer"
        properties:
          name: wordpress-server
          flavor: 4GB Standard Instance
          image: Windows Server 2012
          metadata: {build_config: backup_agent_only}
          user_data:
            str_replace:
              template: |
                $source = "http://download.microsoft.com/download/7/0/4/704CEB4C-9F42-4962-A2B0-5C84B0682C7A/WebPlatformInstaller_amd64_en-US.msi"
                $destination = "webpi.msi"
                $wc = New-Object System.Net.WebClient
                $wc.DownloadFile($source, $destination)
                Start-Process msiexec -ArgumentList "/i webpi.msi /qn"  -NoNewWindow -Wait
                echo DBPassword[@]%dbpassword% DBAdminPassword[@]%dbadminpassword% > test.app
                $tmpprofile = $env:userprofile
                $env:userprofile = "c:\users\administrator"
                $wpicmd = "C:\Program Files\Microsoft\Web Platform Installer\WebPICMD.exe"
                Start-Process $wpicmd -ArgumentList "/Install /Application:Wordpress@test.app /MySQLPassword:%dbadminpassword% /AcceptEULA /Log:.\wpi.log"  -NoNewWindow -Wait
                $env:userprofile = $tmpprofile
              params:
                "%dbpassword%": testpassword_123
                "%dbadminpassword%": testpassword_123

The above resource creates a Windows server and installs the WordPress
application. Please note that '{build_config: backup_agentonly}' was
passed as metadata to install a cloud backup agent.

Cloud backup config resource
----------------------------

Add a Rackspace::Cloud::BackupConfig resource to back up the WordPress
application installed at c:\\inetpub\\wwwroot\\wordpress folder.

.. code:: yaml

      rax_backup_config:
        properties:
          BackupConfigurationName: wordpress-daily-backup
          DayOfWeekId: null
          Frequency: Daily
          StartTimeHour: 11
          StartTimeMinute: 30
          StartTimeAmPm: PM
          HourInterval: 1
          IsActive: true
          Enabled: true
          NotifyFailure: true
          NotifyRecipients: your_email@emailprovider.com
          NotifySuccess: false
          TimeZoneId: Eastern Standard Time
          VersionRetention: 60
          host_ip_address: { get_attr: [wordpress_server, accessIPv4] }
          Inclusions:
          - {"FilePath": "c:\\inetpub\\wwwroot\\wordpress", "FileItemType": "Folder" }
        type: Rackspace::Cloud::BackupConfig

In the above backup resource, the cloud backup service was configured
to create a backup of the 'c:\\inetpub\\wwwroot\\wordpress' folder
'Daily' at '11:30PM' and to retain the created backup for '60'
days. Also, it was configured to notify at the given email ID upon any
error during the backup creation. Please note that ``host_ip_address`` is
the IP address of the cloud server from where files/folders will be backed
up. Here the IP address of the Windows server that was created in the
earlier resource example was passed. If the server was created outside
of the stack, make sure that a backup agent was installed on that
server and pass the IP address to ``host_ip_address``.

Outputs section
---------------

Add the WordPress website URL to the outputs section.

.. code:: yaml

      website_url:
        value:
          str_replace:
            template: http://%ip%/wordpress
            params:
              "%ip%": { get_attr: [ wordpress_server, accessIPv4 ] }
        description: URL for Wordpress site

Full Example Template
---------------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      HEAT template for installing Wordpress on Windows Server

    resources:

      rax_backup_config:
        properties:
          BackupConfigurationName: wordpressbackup
          DayOfWeekId: null
          Frequency: Daily
          StartTimeHour: 7
          StartTimeMinute: 30
          StartTimeAmPm: PM
          HourInterval: null
          IsActive: true
          Enabled: true
          NotifyFailure: true
          NotifyRecipients: vijendar.komalla@rackspace.com
          NotifySuccess: true
          TimeZoneId: Eastern Standard Time
          VersionRetention: 60
          host_ip_address: { get_attr: [rs_windows_server, accessIPv4] }
          Inclusions:
          - {"FilePath": "c:\\inetpub\\wwwroot\\wordpress", "FileItemType": "Folder" }
        type: Rackspace::Cloud::BackupConfig

      rs_windows_server:
        type: "Rackspace::Cloud::WinServer"
        properties:
          name: wordpress-server
          flavor: 4GB Standard Instance
          image: Windows Server 2012
          metadata: {build_config: backup_agent_only}
          user_data:
            str_replace:
              template: |
                $source = "http://download.microsoft.com/download/7/0/4/704CEB4C-9F42-4962-A2B0-5C84B0682C7A/WebPlatformInstaller_amd64_en-US.msi"
                $destination = "webpi.msi"
                $wc = New-Object System.Net.WebClient
                $wc.DownloadFile($source, $destination)
                Start-Process msiexec -ArgumentList "/i webpi.msi /qn"  -NoNewWindow -Wait
                echo DBPassword[@]%dbpassword% DBAdminPassword[@]%dbadminpassword% > test.app
                $tmpprofile = $env:userprofile
                $env:userprofile = "c:\users\administrator"
                $wpicmd = "C:\Program Files\Microsoft\Web Platform Installer\WebPICMD.exe"
                Start-Process $wpicmd -ArgumentList "/Install /Application:Wordpress@test.app /MySQLPassword:%dbadminpassword% /AcceptEULA /Log:.\wpi.log"  -NoNewWindow -Wait
                $env:userprofile = $tmpprofile
              params:
                "%dbpassword%": testpassword_123
                "%dbadminpassword%": testpassword_123

    outputs:
      website_url:
        value:
          str_replace:
            template: http://%ip%/wordpress
            params:
              "%ip%": { get_attr: [ rs_windows_server, accessIPv4 ] }
        description: URL for Wordpress site

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <https://developer.rackspace.com/docs/cloud-orchestration/v1/developer-guide/>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud-init format
   documentation <http://cloudinit.readthedocs.org/en/latest/topics/format.html>`__
-  `Cloud backup getting started
   guide <https://developer.rackspace.com/docs/cloud-backup/v1/getting-started/>`__
-  `Cloud backup API developer
   guide <https://developer.rackspace.com/docs/cloud-backup/v1/api-reference/>`__
-  `Install cloud backup agent on Linux
   server <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-linux>`__
-  `Install cloud backup agent on Windows
   server <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-windows>`__
-  `Install cloud backup agent on Windows server (silent
   installation) <http://www.rackspace.com/knowledge_center/article/rackspace-cloud-backup-install-the-agent-on-windows-by-using-silent-installation>`__

