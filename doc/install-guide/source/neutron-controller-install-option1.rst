Networking Option 1: Provider networks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install and configure the Networking components on the *controller* node.

Prerequisites
-------------

Before you configure networking option 1, you must configure kernel
parameters to disable reverse-path filtering.

#. Edit the ``/etc/sysctl.conf`` file to contain the following parameters:

   .. code-block:: ini

      net.ipv4.conf.all.rp_filter=0
      net.ipv4.conf.default.rp_filter=0

#. Implement the changes:

   .. code-block:: console

      # sysctl -p

Install the components
----------------------

.. only:: ubuntu

   .. code-block:: console

      # apt-get install neutron-server neutron-plugin-ml2 \
        neutron-plugin-linuxbridge-agent neutron-dhcp-agent \
        neutron-metadata-agent python-neutronclient

.. only:: rdo

   .. code-block:: console

      # yum install openstack-neutron openstack-neutron-ml2 \
        openstack-neutron-linuxbridge python-neutronclient

.. only:: obs

   .. code-block:: console

      # zypper install --no-recommends openstack-neutron \
        openstack-neutron-server openstack-neutron-linuxbridge-agent \
        openstack-neutron-dhcp-agent openstack-neutron-metadata-agent \
        ipset

.. only:: debian

   Install and configure the networking components
   -----------------------------------------------

   #. .. code-block:: console

         # apt-get install neutron-server neutron-plugin-linuxbridge-agent \
           neutron-dhcp-agent neutron-metadata-agent

      For networking option 2, also install the ``neutron-l3-agent`` package.

   #. Respond to prompts for `database
      management <#debconf-dbconfig-common>`__, `Identity service
      credentials <#debconf-keystone_authtoken>`__, `service endpoint
      registration <#debconf-api-endpoints>`__, and `message queue
      credentials <#debconf-rabbitmq>`__.

   #. Select the ML2 plug-in:

      .. image:: figures/debconf-screenshots/neutron_1_plugin_selection.png

      .. note::

         Selecting the ML2 plug-in also populates the ``service_plugins`` and
         ``allow_overlapping_ips`` options in the
         ``/etc/neutron/neutron.conf`` file with the appropriate values.

.. only:: ubuntu or rdo or obs

   Configure the server component
   ------------------------------

   The Networking server component configuration includes the database,
   authentication mechanism, message queue, topology change notifications,
   and plug-in.

   .. include:: shared/note_configuration_vary_by_distribution.rst

   #. Edit the ``/etc/neutron/neutron.conf`` file and complete the following
      actions:

      * In the ``[database]`` section, configure database access:

        .. code-block:: ini

           [database]
           ...
           connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

        Replace ``NEUTRON_DBPASS`` with the password you chose for the
        database.

      * In the ``[DEFAULT]`` section, enable the Modular Layer 2 (ML2)
        plug-in and disable additional plug-ins:

        .. code-block:: ini

           [DEFAULT]
           ...
           core_plugin = ml2
           service_plugins =

      * In the ``[DEFAULT]`` and ``[oslo_messaging_rabbit]`` sections,
        configure RabbitMQ message queue access:

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
        ``openstack`` account in RabbitMQ.

      * In the ``[DEFAULT]`` and ``[keystone_authtoken]`` sections, configure
        Identity service access:

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
           username = neutron
           password = NEUTRON_PASS

        Replace ``NEUTRON_PASS`` with the password you chose for the ``neutron``
        user in the Identity service.

        .. note::

           Comment out or remove any other options in the
           ``[keystone_authtoken]`` section.

      * In the ``[DEFAULT]`` and ``[nova]`` sections, configure Networking to
        notify Compute of network topology changes:

        .. code-block:: ini

           [DEFAULT]
           ...
           notify_nova_on_port_status_changes = True
           notify_nova_on_port_data_changes = True
           nova_url = http://controller:8774/v2

           [nova]
           ...
           auth_url = http://controller:35357
           auth_plugin = password
           project_domain_id = default
           user_domain_id = default
           region_name = RegionOne
           project_name = service
           username = nova
           password = NOVA_PASS

        Replace ``NOVA_PASS`` with the password you chose for the ``nova``
        user in the Identity service.

      * (Optional) To assist with troubleshooting, enable verbose logging in
        the ``[DEFAULT]`` section:

        .. code-block:: ini

           [DEFAULT]
           ...
           verbose = True

Configure the Modular Layer 2 (ML2) plug-in
-------------------------------------------

The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging
and switching) virtual networking infrastructure for instances.

#. Edit the ``/etc/neutron/plugins/ml2/ml2_conf.ini`` file and complete the
   following actions:

   * In the ``[ml2]`` section, enable flat and VLAN networks:

     .. code-block:: ini

        [ml2]
        ...
        type_drivers = flat,vlan

   * In the ``[ml2]`` section, disable project (private) networks:

     .. code-block:: ini

        [ml2]
        ...
        tenant_network_types =

   * In the ``[ml2]`` section, enable the Linux bridge mechanism:

     .. code-block:: ini

        [ml2]
        ...
        mechanism_drivers = linuxbridge

     .. warning::

        After you configure the ML2 plug-in, removing values in the
        ``type_drivers`` option can lead to database inconsistency.

   * In the ``[ml2]`` section, enable the port security extension driver:

     .. code-block:: ini

        [ml2]
        ...
        extension_drivers = port_security

   * In the ``[ml2_type_flat]`` section, configure the public flat provider
     network:

     .. code-block:: ini

        [ml2_type_flat]
        ...
        flat_networks = public

Configure the Linux bridge agent
--------------------------------

The Linux bridge agent builds layer-2 (bridging and switching) virtual
networking infrastructure for instances including VXLAN tunnels for private
networks and handles security groups.

#. Edit the ``/etc/neutron/plugins/ml2/linuxbridge_agent.conf`` file and
   complete the following actions:

   * In the ``[linux_bridge]`` section, map the public virtual network to the
     public physical network interface:

     .. code-block:: ini

       [linux_bridge]
       physical_interface_mappings = public:PUBLIC_INTERFACE_NAME

     Replace ``PUBLIC_INTERFACE_NAME`` with the name of the underlying physical
     public network interface.

   * In the ``[vxlan]`` section, disable VXLAN overlay networks:

     .. code-block:: ini

        [vxlan]
        enable_vxlan = False

   * In the ``[agent]`` section, enable ARP spoofing protection:

     .. code-block:: ini

        [agent]
        ...
        prevent_arp_spoofing = True

   * In the ``[securitygroup]`` section, enable security groups, enable
     :term:`ipset`, and configure the Linux bridge :term:`iptables` firewall
     driver:

     .. code-block:: ini

        [securitygroup]
        ...
        enable_security_group = True
        enable_ipset = True
        firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

Configure the DHCP agent
------------------------

The :term:`DHCP agent` provides DHCP services for virtual networks.

#. Edit the ``/etc/neutron/dhcp_agent.ini`` file and complete the following
   actions:

   * In the ``[DEFAULT]`` section, configure the Linux bridge interface driver,
     Dnsmasq DHCP driver, and enable isolated metadata so instances on public
     networks can access metadata over the network:

     .. code-block:: ini

        [DEFAULT]
        ...
        interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        enable_isolated_metadata = True

   * (Optional) To assist with troubleshooting, enable verbose logging in the
     ``[DEFAULT]`` section:

     .. code-block:: ini

        [DEFAULT]
        ...
        verbose = True

Return to
:ref:`Networking controller node configuration
<neutron-controller-metadata-agent>`.
