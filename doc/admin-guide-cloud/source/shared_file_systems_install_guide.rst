================================================
Install and configure shared filesystem services
================================================

This section describes how to install and configure the Shared
Filesystems service, code-named manila, on the controller node.

.. note::

   In this example, ``manila-api``, ``manila-scheduler`` and ``manila-share``
   are all installed on the same node.


To configure prerequisites
~~~~~~~~~~~~~~~~~~~~~~~~~~

Before you install and configure the Shared Filesystems service, you
must create a database, service credentials, and API endpoint.

#. To create the database, complete these steps:

   a. Use the database access client to connect to the database
      server as the ``root`` user:

      .. code-block:: console

         $ mysql -u root -p

   b. Create the ``manila`` database:

      .. code-block:: console

         CREATE DATABASE manila CHARACTER SET utf8;

   c. Grant proper access to the ``manila`` database:

      .. code-block:: console

         GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'localhost' \
           IDENTIFIED BY 'MANILA_DBPASS';
         GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'%' \
           IDENTIFIED BY 'MANILA_DBPASS';

      Replace ``MANILA_DBPASS`` with a suitable password.

   d. Exit the database access client.

#. Source the ``admin`` credentials to gain access to admin-only
   CLI commands:

   .. code-block:: console

      $ source admin-openrc.sh

#. To create the service credentials, complete these steps:

   a. Create a ``manila`` user:

      .. code-block:: console

         $ openstack user create --password-prompt manila
         User Password:
         Repeat User Password:
         +----------+----------------------------------+
         | Field    | Value                            |
         +----------+----------------------------------+
         | email    | None                             |
         | enabled  | True                             |
         | id       | 881ab2de4f7941e79504a759a83308be |
         | name     | manila                           |
         | username | manila                           |
         +----------+----------------------------------+

   b. Add the ``admin`` role to the ``manila`` user:

      .. code-block:: console

         $ openstack role add --project service --user manila admin
         +-------+----------------------------------+
         | Field | Value                            |
         +-------+----------------------------------+
         | id    | cd2cb9a39e874ea69e5d4b896eb16128 |
         | name  | admin                            |
         +-------+----------------------------------+

   c. Create the ``manila`` service entity:

      .. code-block:: console

         $ openstack service create --name manila \
           --description "OpenStack Shared Filesystems" share
         +-------------+----------------------------------+
         | Field       | Value                            |
         +-------------+----------------------------------+
         | description | OpenStack Shared Filesystems     |
         | enabled     | True                             |
         | id          | 1e494c3e22a24baaafcaf777d4d467eb |
         | name        | manila                           |
         | type        | share                            |
         +-------------+----------------------------------+

#. Create the Shared Filesystems service API endpoints:

   .. code-block:: console

      $ openstack endpoint create \
        --publicurl http://controller:8786/v1/%\(tenant_id\)s \
        --internalurl http://controller:8786/v1/%\(tenant_id\)s \
        --adminurl http://controller:8786/v1/%\(tenant_id\)s \
        --region RegionOne \
        share
      +--------------+-----------------------------------------+
      | Field        | Value                                   |
      +--------------+-----------------------------------------+
      | adminurl     | http://controller:8786/v1/%(tenant_id)s |
      | id           | d1b7291a2d794e26963b322c7f2a55a4        |
      | internalurl  | http://controller:8786/v1/%(tenant_id)s |
      | publicurl    | http://controller:8786/v1/%(tenant_id)s |
      | region       | RegionOne                               |
      | service_id   | 1e494c3e22a24baaafcaf777d4d467eb        |
      | service_name | manila                                  |
      | service_type | share                                   |
      +--------------+-----------------------------------------+

To install and configure Shared Filesystems components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. only:: obs

   1. Install the packages:

      .. code-block:: console

         # zypper install openstack-manila-api openstack-manila-scheduler \
                      openstack-manila-share python-manilaclient

      .. note::

         Packages for different openSUSE and SLE releases for different
         OpenStack versions can be found on the
         `OpenBuildService <https://build.opensuse.org/project/subprojects/Cloud:OpenStack/>`_ .

.. only:: rdo

   1. Install the packages:

      .. code-block:: console

         # yum install openstack-manila python-manilaclient

.. only:: ubuntu

   1. Install the packages:

      .. code-block:: console

         # apt-get install manila-api manila-scheduler python-manilaclient

2.

   Edit the :file:`/etc/manila/manila.conf` file and complete the
   following actions:

   a. In the ``[database]`` section, configure database access:

      .. code-block:: ini

         [database]
         ...
         connection = mysql://manila:MANILA_DBPASS@controller/manila

      Replace ``MANILA_DBPASS`` with the password you chose for the
      Shared Filesystems database.

   b. In the ``[DEFAULT]`` and ``[oslo_messaging_rabbit]`` sections,
      configure ``RabbitMQ`` message queue access:

      .. code-block:: ini

         [DEFAULT]
         ...
         rpc_backend = rabbit

         [oslo_messaging_rabbit]
         ...
         rabbit_host = controller
         rabbit_userid = openstack
         rabbit_password = RABBIT_PASS

      Replace ``RABBIT_PASS`` with the password you chose for the
      ``openstack`` account in ``RabbitMQ``.

   c. In the ``[DEFAULT]`` and ``[keystone_authtoken]`` sections,
      configure Identity service access:

      .. code-block:: ini

         [DEFAULT]
         ...
         auth_strategy = keystone

         [keystone_authtoken]
         ...
         auth_uri = http://controller:5000
         auth_url = http://controller:35357
         auth_plugin = password
         project_domain_id = default
         user_domain_id = default
         project_name = service
         username = manila
         password = MANILA_PASS

      Replace ``MANILA_PASS`` with the password you chose for
      the ``manila`` user in the Identity service.

   d. In the ``[DEFAULT]`` section, configure the ``my_ip`` option to
      use the management interface IP address of the controller node:

      .. code-block:: ini

         [DEFAULT]
         ...
         my_ip = 10.0.0.11

   e. In the ``[oslo_concurrency]`` section, configure the lock path:

      .. code-block:: ini

         [oslo_concurrency]
         ...
         lock_path = /var/lock/manila

   f. Setup access to Nova, Cinder and Neutron in the ``[DEFAULT]`` section:

      .. code-block:: ini

         [DEFAULT]
         ...
         nova_admin_password = NOVA_PASS
         nova_admin_tenant_name = service
         cinder_admin_password = CINDER_PASS
         cinder_admin_tenant_name = service
         neutron_admin_password = NEUTRON_PASS
         neutron_admin_project_name = service
         verbose = True

      Replace ``NOVA_PASS``, ``CINDER_PASS`` and ``NEUTRON_PASS``
      with the passwords you chose for the users in the Identity service.

   g. Configure a backend.
      In this example, we use the generic driver as backend. Configure the
      ``[backend-1]`` section and enable the backend in the ``[DEFAULT]``
      section:

      .. code-block:: ini

         [DEFAULT]
         ...
         enabled_share_backends = backend-1

      .. code-block:: ini

         [backend-1]
         share_driver = manila.share.drivers.generic.GenericShareDriver
         driver_handles_share_servers = True
         service_instance_user = IMAGE_USER
         service_instance_password = IMAGE_PASSWORD
         service_image_name = manila-service-image
         share_backend_name = Backend1

      Replace ``IMAGE_USER`` and ``IMAGE_PASSWORD`` with the credentials from
      the used ``service_image_name``.

      .. note::

         The generic share driver uses a image from the Image service.
         This image includes an NFS and CIFS server to be able to export
         filesystems. Other drivers may not need a specific image.
         In this example, the used image name is ``manila-service-image``.
         There's a diskimage-builder project to create custom images called
         `manila-image-elements <https://github.com/openstack/manila-image-elements>`_ .

      .. only:: obs

        .. note::
           Custom images for openSUSE and SLES can also be built with
           `SUSE Studio <https://susestudio.com/>`_ .

   g. (Optional) To assist with troubleshooting, enable verbose
      logging in the ``[DEFAULT]`` section:

      .. code-block:: ini

         [DEFAULT]
         ...
         verbose = True

3. Populate the Shared Filesystems database:

   .. code-block:: console

      # su -s /bin/sh -c "manila-manage db sync" manila

To finalize installation
~~~~~~~~~~~~~~~~~~~~~~~~

.. only:: obs or rdo

   1. Start the Shared Filesystems services and configure them to start when
      the system boots:


      .. code-block:: console

         # systemctl enable openstack-manila-api.service openstack-manila-scheduler.service \
                      openstack-manila-share.service
         # systemctl start openstack-manila-api.service openstack-manila-scheduler.service \
                      openstack-manila-share.service

.. only:: ubuntu

   1. Restart the Shared Filesystems services:

      .. code-block:: console

         # service manila-scheduler restart
         # service manila-api restart
         # service manila-share restart

   2. By default, the Ubuntu packages create an SQLite database.

      Because this configuration uses a SQL database server,
      you can remove the SQLite database file:

      .. code-block:: console

         # rm -f /var/lib/manila/manila.sqlite


============================
Verify and testing operation
============================

This section describes how to verify operation of the Shared Filesystems
service by creating a share.

.. note::

   Perform these commands on the controller node.

#. Source the ``admin`` credentials to gain access to
   admin-only CLI commands:

   .. code-block:: console

      $ source admin-openrc.sh

#. List service components to verify successful launch of each process:

   .. code-block:: console

      $ manila service-list
      +----+------------------+------------------------------+------+---------+-------+----------------------------+
      | Id | Binary           | Host                         | Zone | Status  | State | Updated_at                 |
      +----+------------------+------------------------------+------+---------+-------+----------------------------+
      | 1  | manila-scheduler | controller                   | nova | enabled | up    | 2015-08-17T15:16:04.372185 |
      | 2  | manila-share     | controller@backend-generic-0 | nova | enabled | up    | 2015-08-17T15:16:03.324475 |
      +----+------------------+------------------------------+------+---------+-------+----------------------------+

#. Create a default share type:

   .. code-block:: console

      $ manila type-create default_share_type True
      +--------------------------------------+--------------------+------------+------------+-------------------------------------+
      | ID                                   | Name               | Visibility | is_default | required_extra_specs                |
      +--------------------------------------+--------------------+------------+------------+-------------------------------------+
      | 1536b47b-2ed5-4949-9dd7-9962f9b66df4 | default_share_type | public     | -          | driver_handles_share_servers : True |
      +--------------------------------------+--------------------+------------+------------+-------------------------------------+

#. Create a manila share network:
To operate a driver that handles share servers (like the Generic driver), we
must create a share network, which is a set of network information that will be
used during share server creation. In our example, to use Neutron, we will do
the following:

    .. code-block:: console

       $ neutron net-list

Here we note the ID of a Neutron network and one of its subnets.

   .. note::

      Some configurations of the Generic driver may require this network be
      attached to a public router. It is so by default. So, if you use the
      default configuration of Generic driver, make sure the network is
      attached to a public router.

   .. code-block:: console

      $ manila share-network-create \
      --name test_share_network \
      --neutron-net-id %id_of_neutron_network% \
      --neutron-subnet-id %id_of_network_subnet%

#. Create a share and allow access

   .. code-block:: console

      $ manila create NFS 1 --name testshare --share-network test_share_network
      $ manila access-allow testshare ip 0.0.0.0/0 --access-level rw