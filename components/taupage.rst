.. _taupage:

=======
Taupage
=======

**Taupage** is the base AMI allowing dockerized applications to run with STUPS.

As we want to foster immutable (and therefore deterministic and reproducible) deployments, we want to encourage the use
of Docker (and similar deployment technologies). The Taupage AMI is capable of starting a Docker container on boot. This
will enable teams to deploy 'what they want' as long as they package it in a Docker image. The server will be
set up to have an optimal configuration including managed SSH access, audit logging, log collection, monitoring and
reviewed security additions.

Using the Taupage AMI
++++++++++++++++++++++

There is currently no internal tooling but you can find the Taupage AMIs in your EC2 UI. They are maintained by the
STUPS team and regularly updated with the most recent security fixes and configuration improvements.

.. NOTE::
   The process of updating the AMI is not established nor discussed yet!

How to configure the AMI (configuration example)
++++++++++++++++++++++++++++++++++++++++++++++++

The Taupage AMI uses the official cloud-init project to receive user configuration. Different to the standard, you can
not use the normal user data mimetypes (no #cloud-config, shell scripts, file uploads, URL lists, ...) but only our own
configuration format::

   #taupage-ami-config

   application_id: my-nginx-test-app
   application_version: "1.0"

   runtime: Docker
   source: "pierone.example.org/myteam/nginx:1.0"

   dockercfg:
     "https://hub.docker.com":
       auth: foo1234
       email: mail@example.org

   ports:
     80: 80
     443: 443
     8301: 8301
     8301/udp: 8301
     8600: 8600/upd

   health_check_port: 80
   health_check_path: /
   health_check_timeout_seconds: 60

   environment:
     STAGE: production
     # environment variable values starting with "aws:kms:"
     # automatically are decrypted by Taupage
     MY_DB_PASSWORD: "aws:kms:v5V2bMGRgg2yTHXm5Fn..."

   capabilities_add:
     - NET_BIND_SERVICE
   capabilities_drop:
     - NET_ADMIN

   root: false
   privileged: false
   docker_daemon_access: false
   read_only: false
   mount_var_log: false
   mount_certs: false
   keep_instance_users: false
   enhanced_cloudwatch_metrics: true

   volumes:
     ebs:
       # attach EBS volume with "Name" tag "foo"
       /dev/sdf: foo
       # attach EBS volume with "Name" tag "bar"
       /dev/sdg: bar

     raid:
       # Defines RAID0 volume with the attached devices above (note the different device names)
       /dev/md/sampleraid0:
         level: 0
         devices:
           - /dev/xvdf
           - /dev/xvdg

   mounts:
     # Define a mountpoint for the above RAID volume which should be re-used without reformatting
     /some_volume:
       partition: /dev/md/sampleraid0
       erase_on_boot: false
       filesystem: ext4 # Default filesystem is ext4

     # An example for a non RAID configuration, which mounts regular devices on your EC2 instance
     /data:
       partition: /dev/xvdb
       erase_on_boot: true
     /data1:
       partition: /dev/xvdc
       filesystem: ext3

   notify_cfn:
     stack: pharos
     resource: WebServerGroup

   # configure cloudwatch logs agent (logfile --> log-group mapping)
   cloudwatch_logs:
     /var/log/syslog: my-syslog-loggroup
     /var/log/application.log: my-application-loggroup

   ssh_ports:
     - 22

   ssh_gateway_ports: no

   etcd_discovery_domain: etcd.myteam.example.org

   logentries_account_key: 12345-ACCOUNT-12345-KEY
   # AWS KMS encryption available. Example:
   logentries_account_key: "aws:kms:v5V2bMGRgg2yTHXm5Fn..."

   scalyr_account_key: 12345-ACCOUNTKEY-12234
   # AWS KMS encryption available. Example:
   scalyr_account_key: "aws:kms:v5V2bMGRgg2yTHXm5Fn..."
   scalyr_application_log_parser: customParser

   newrelic_account_key: 12345-ACCOUNTKEY-12234

   mint_bucket: my-s3-mint-bucket

   #configure logrotate for application.log
   application_logrotate_size: 10M
   application_logrotate_interval: daily
   application_logrotate_rotate: 4
   


Provide this configuration as your user-data during launch of your EC2 instance.
You can use the ``TaupageConfig`` section of :ref:`senza`'s ``TaupageAutoScalingGroup``
to easily pass Taupage options when deploying with Senza.

Configuration option explanation
++++++++++++++++++++++++++++++++

application_id:
---------------

**(required)**

The well-known, registered (in :ref:`kio`) application identifier/name. Examples: "order-engine", "eventlog-service", ..

application_version:
--------------------

**(required)**

The well-known, registered (in :ref:`kio`) application version string. Examples: "1.0", "0.1-alpha", ..

runtime:
--------

**(required)**

What kind of deployment artifact you are using. Currently supported:

* Docker

.. NOTE::
   We plan to integrate CoreOS's Rocket as a runtime for experimental use soon.

source:
-------

**(required)**

The source, the configured runtime uses to fetch your deployment artifact. For Docker, this is the Docker image.
Usually this will point to a Docker image stored in :ref:`pierone`.

.. NOTE::
   If the registry part of source contains 'pierone':
     Taupage assumes it needs to pull the image from Pierone and uses OAuth2 credentials of the application set in **application_id** to authenticate the download of the (Docker) image. This requires a Mint/Berry setup and Pierone indeed.
   If there is a dockercfg config key in the taupage.yaml:
     Taupage uses the credentials from dockercfg to do basic auth against a registry.
   If there is neither pierone nor dockercfg:
     Taupage will not try to authenticate the download.

dockercfg:
----------

**(optional)**

The intended content of ~/.dockercfg on a Taupage instance. This allows to configure authentication for non-Pierone registries which require basic auth.
The following example shows a configuration for private docker hub protected with basic auth. 'auth' must contain a base64 encoded string in '<user>:<password>' format.

Example:
  dockercfg:
    "https://hub.docker.com":
      auth: <base64 encoded user:password>

      email: mail@example.org

ports:
------

**(optional, default: no ports open)**

A map of all ports that have to be opened from the container. The key is the public server port to open and its value is the original port in your container. By default only TCP ports are opened. If you want to open UDP ports, you have to specify UDP protocol as a part of value or key::

   ports:
     8301: 8301  # open 8301 tcp port
     8301/udp: 8301  # open 8301 udp port
     8600: 8600/upd  # open 8600 udp port

health_check_path:
------------------

**(optional)**

HTTP path to check for status code 200. Taupage will wait at most ``health_check_timeout_seconds`` (default: 60) until the health check endpoint becomes OK.
The health check port is using the first port from ``ports`` or can be overwritten with ``health_check_port``.


environment:
------------

**(optional)**

A map of environment variables to set. Environment variable values starting with "aws:kms:" are automatically decrypted by Taupage using KMS (IAM role needs to allow decryption with the used KMS key).

To create a key on kms see `here <http://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html>`_.
After this, `install the kmsclient <https://github.com/zalando/kmsclient>`_ and follow the instructions to encrypt a value using the created key.
Following this, add the encrypted value to the environment variable in the format "aws:kms:<encrypted_value>"

Example::

    environment:
      STAGE: production
      # environment variable values starting with "aws:kms:"
      # automatically are decrypted by Taupage
      MY_DB_PASSWORD: "aws:kms:v5V2bMGRgg2yTHXm5Fn..."

capabilities_add:
-----------------

**(optional)**

A list of capabilities to add to the execution (without the CAP_ prefix). See
http://man7.org/linux/man-pages/man7/capabilities.7.html for available capabilities.

capabilities_drop:
------------------

**(optional)**

A list of capabilities to drop of the execution (without the CAP_ prefix). See
http://man7.org/linux/man-pages/man7/capabilities.7.html for available capabilities.

hostname:
---------

**(optional)**

TBD, Users can define hostname by themselves

networking:
-----------

**(optional)**

A type of networking to tell how docker networks a container. See
https://docs.docker.com/articles/networking/#how-docker-networks-a-container for details.

Options are:
  * bridge (default)
  * host (This option also passes the hostname/instance name to the Docker container)
  * container:NAME_or_ID
  * none

root:
-----

**(optional, default: false)**

Specifies, if the container has to run as root. By default, containers run as an unprivileged user. See the
**capabilities_add** and prefer it always. This is only the last resort.

privileged:
-----------

**(optional, default: false)**

The container will run with --privileged option.
See https://docs.docker.com/reference/run/#runtime-privilege-linux-capabilities-and-lxc-configuration for more details.
**Warning: this has serious security implications that you must understand and consider!**

docker_daemon_access:
-----------

**(optional, default: false)**

Mount the /var/run/docker.sock into the running container. This way, you are able to use and control
the Docker daemon of the host system. **Warning: this has serious security implications that you must
understand and consider!**

read_only:
-----------

**(optional, default: false)**

The container will run with --read-only option.
Mount the container's root filesystem as read only.

mount_var_log:
-----------

**(optional, default: false)**

This will mount /var/log into the Docker container as read-only.

mount_certs:
-----------

**(optional, default: false)**

This will mount /etc/ssl/certs into the Docker container as read-only.

keep_instance_users: true:
--------------------

**(optional, default: false)**

This option allows you to keep the users on the instance, created by AWS.
The ubuntu user, it's authorized_keys and the root users authorized_keys will be deleted.
Access to the instances will be granted via Even&Odd.
See https://docs.stups.io/en/latest/user-guide/ssh-access.html for more.

enhanced_cloudwatch_metrics: true
--------------------

**(optional, default: false)**

This option allows you to enable enhanced Cloudwatch metrics, such as memory and disk space, which are out of the box not enabled.

.. NOTE::
   This requires the AWS IAM policy "cloudwatch:PutMetricData".

volumes:
--------

**(optional)**

Allows you to configure volumes that can later be mounted. Volumes accepts two sub-configurations - **EBS** and **RAID**.

EBS
^^^

The EBS sub-configuration expects key-value pairs of device name to EBS volumes. The "Name" tag is used to find the volumes.

Sample EBS volume configuration::

     ebs:
       /dev/sdf: solr-repeater-volume
       /dev/sdg: backup-volume


.. NOTE::
   You also have to create a **IAM Role** for this. Resource can be "*" or the ARN of the Volume ( arn:aws:ec2:region:account:volume/volume-id ).



IAM-Role::

     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Sid": "TaupageVolumeAccess",
           "Effect": "Allow",
           "Action": [
               "ec2:AttachVolume",
               "ec2:DescribeVolumes"
           ],
           "Resource": [
               "*"
           ]
         }
       ]
     }

RAID
^^^^

The RAID sub-configuration allows you to describe RAID volumes by specifying the device name, usually */dev/md/your-raid-name*, and
all of the required RAID definitions.

You need to provide the RAID **level** and a collection of, at least, 2 **devices** to build your
RAID volume. The amount of devices is dependent on the RAID level. See http://en.wikipedia.org/wiki/Standard_RAID_levels#Comparison

Sample RAID volume configuration::

     raid:
       /dev/md/solr-repeater:
         level: 5
         devices:
           - /dev/xvdf
           - /dev/xvdg
           - /dev/xvdh

.. NOTE::
   EBS volumes are always attached first. This way you can use them in your RAID definitions.
   But it doesn't necessarily makes sense to use them in a RAID configuration, since AWS already mirrors them internally.

   Depending on your instance virtualisation type, the final device names can be slightly different. Please refer to:

       * `AWS EC2 Block Device Mapping <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/block-device-mapping-concepts.html>`_
       * `AWS EC2 Device Naming on Linux Instances <http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/device_naming.html>`_

mounts:
-------

**(optional)**

A map of mount targets and their configurations. A mount target configuration has a **partition** to reference the volume, which can be
defined in the **volumes** section. It is possible to specify a **erase_on_boot** flag which determines is such partition should always
be initialized on boot. This setting defaults to false.

Whenever a partition is initialized is will be formatted using the **filesystem** setting. If unspecified it will be formatted as ext4. If **options** setting is specified, its value will be provided to the command to mount the partition. If the **root** setting is false (that's the default) the filesystem will be initialized with the internal unprivileged user as its owner. The mount point permissions are set to provide read and write access to group and others in all cases. This allows the **runtime** application to use the volume for read and write.

Sample mounts configuration::

   mounts:
     /data/solr:
       partition: /dev/md/solr-repeater
       options: noatime,nodiratime,nobarrier
       erase_on_boot: false

.. WARNING::
   Volumes that are supposed to be re-used (and not follow the instance lifecycle) should be initialized in some other way, for ex., attaching it to another running instance and performing the required steps there only once.
   If the **erase_on_boot** flag is True it will always be initialized. Trying to reuse EBS volumes with this flag is a bit useless.


notify_cfn:
-----------

**(optional)**

Will send cloud formation the boot result if specified. If you specify it, you have to provide the **stack** name and
the stack **resource** with which this server was booted. This helps cloud formation to know, if starting you server
worked or not (else, it will run into a timeout, waiting for notifications to arrive).

If you would use the example stack
http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/example-templates-autoscaling.html
the resource name would be **WebServerGroup**.

cloudwatch_logs:
----------------

**(optional)**

Will configure the awslogs agent to stream logfiles to AWS Cloudwatch Logs service. One needs to define a mapping of logfiles to their destination loggroups.
There will be a stream for each instance in each configured logfile/loggroup.

Documentation for Cloudwatch Logs:
http://docs.aws.amazon.com/AmazonCloudWatch/latest/DeveloperGuide/WhatIsCloudWatchLogs.html

Example:
   cloudwatch_logs:
     /var/log/application.log: my-application-loggroup

Will configure the awslogs daemon to stream the /var/log/application.log file into the my-application-loggroup.

ssh_ports:
----------

**(optional, default: 22)**

List of SSH server ports. This option allows using alternative TCP ports for the OpenSSH server.
This is useful if an application (runtime container) wants to use the default SSH port.

ssh_gateway_ports:
------------------

**(optional)**

Adds `GatewayPorts` config line to sshd_config which specifies whether remote hosts are allowed to connect to local forwarded ports.
This is useful with value "yes" for example: `GatewayPorts yes` if reverse tunnel to the Taupage instance is needed.

etcd_discovery_domain:
----------------------

**(optional)**

DNS domain for `etcd`_ cluster discovery. Taupage will start a local etcd proxy if the ``etcd_discovery_domain`` is specified.
The proxy's HTTP endpoint is passed in the ``ETCD_URL`` environment variable to the application, i.e. ``curl $ETCD_URL/v2/keys/`` should `list all keys`_.
You need a running `etcd cluster with DNS registration`_ for this option to work.

In the etcd service all running instances will be registered within the key ``taupage``. All Nodes wich have the  ``etcd_discovery_domain`` set will be dynamically added and removed from this entry:

   curl $ETCD_URL/v2/keys/taupage

.. _etcd: https://coreos.com/etcd/
.. _list all keys: https://coreos.com/etcd/docs/latest/api.html#listing-a-directory
.. _etcd cluster with DNS registration: https://github.com/zalando/spilo/tree/master/etcd-cluster-appliance


logentries_account_key:
-----------------------

**(optional)**

.. NOTE::
   You can also use AWS KMS to encrypt your Logentries account key. See in the example above.

If you specify the Account Key from your logentries account, the Logentries Agent will be registered with your Account.
And the Agent will follow these logs:

  * /var/log/syslog
  * /var/log/auth.log
  * /var/log/application.log

You can get your Account Key from the Logentries Webinterface under /Account/Profile


scalyr_account_key
------------------

**(optional)**

.. NOTE::
   You can also use AWS KMS to encrypt your Scalyr account key. See in the example above.

If you provide the Scalyr AccountKey in the .yaml file, the agent of Scaylr will be installed and will follow these logs:

  * /var/log/syslog
  * /var/log/auth.log
  * /var/log/application.log

Our integration also provides some attributes you can search on Scalyr:

  * **$application_id**
  * **$application_version**
  * **$stack**
  * **$source**
  * **$image**

scalyr_application_log_parser
-----------------------------

**(optional)**

If the application.log format differs heavily between multiple applications the parser definition used by Scalyr can be overwritten here. The default value is `slf4j`.

newrelic_account_key
--------------------

**(optional)**

If you provide the NewRelic license key in the .yaml file, the newrelic-sysmond will be set up.

application_logrotate_*
-----------------------

**(optional)**

These are settings how logrotate will rotate your application.log file.


    **examples**::

        application_logrotate_size: 10M
        application_logrotate_interval: weekly
        application_logrotate_rotate: 4

    **explanation**:

  * **application_logrotate_size**
     * Log files are rotated when they grow bigger than size bytes. If size is followed by M, the size if assumed to be in megabytes. If the G suffix is used, the size is in gigabytes. If the k is used, the size is in kilobytes. So size 100, size 100k, and size 100M are all valid.
     * **Default: size is not set**
  * **application_logrotate_interval**
     * the time interval when logs will be rotated: hourly, daily, weekly, monthly, yearly is possible.
     * **Default: weekly**
  * **application_logrotate_rotate**
     * Log files are rotated count times before being removed or mailed to the address specified in a mail directive. If count is 0, old versions are removed rather than rotated.
     * **Default: 4**


Runtime environment
+++++++++++++++++++

By default, your application will run as an unprivileged user, see the 'root' option.

Taupage integrates :ref:`berry` and exposes the credentials file to your application. Your application will have access
to the environment variable 'CREDENTIALS_DIR', which points to a local directory, containing the 'user.json' and 'client.json' of
the :ref:`mint` API. This way, you can authenticate yourself to your own IAM solution so that it can obtain its own access
tokens.

Sending application mails
+++++++++++++++++++++++++

Mails which should be sent from applications can be sent out directly via Amazon SES.
The only thing you need to do is create an IAM user and receive SMTP credentials. This can be done directly in the SES menu.
Amazon already provides an example for Java: http://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-using-smtp-java.html

In order to use SES for sending out mails into the world, you need to request a limit increase (100 = 50k mails/day) to get
your account out of the sandbox mode.

AMI internals
+++++++++++++

This section gives you an overview of customization, the Taupage AMI contains on top of the Ubuntu Cloud Images.

Docker application logging
--------------------------

Application logs by Docker containers are streamed to syslog via Docker's logging driver for syslog as described
in the Docker documentation: https://docs.docker.com/reference/run/#logging-driver-syslog

Managed SSH access
------------------

SSH access is managed with the :ref:`even` SSH access granting service. The AMI is set up to have automatic integration. Your
SSH key pair choice on AWS will be ignored - temporary access can only be gained via the granting service. All user
actions are logged for auditing reasons. See the :ref:`ssh-access` section in the User's Guide for details.

Building your own AMI
+++++++++++++++++++++

You can build your own Taupage AMI using the code from the repository on GitHub https://github.com/zalando-stups/taupage
In the repository you will find a configuration (config-stups-example.sh) file which you'll have to adjust to your needs.

See :ref:`taupage-ami-creation` for details.
