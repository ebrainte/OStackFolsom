==========================================================
  OpenStack Folsom Install Guide
==========================================================

:Version: 1.0
:Source: https://github.com/mseknibilel/OpenStack-Folsom-Install-guide
:Keywords: Multi node OpenStack, Folsom, Nova, Nova-Network, Keystone, Glance, Cinder, KVM, Ubuntu Server 12.10 (64 bits).

Authors
==========

Copyright (C) Bilel Msekni <bilel.msekni@telecom-sudparis.eu>

Contributors
==========

* Roy Sowa <Roy.Sowa@ssc-spc.gc.ca>
* Marco Consonni <marco_consonni@hp.com>
* Dennis E Miyoshi <dennis.miyoshi@hp.com>
* Houssem Medhioub <houssem.medhioub@it-sudparis.eu>
* Djamal Zeghlache <djamal.zeghlache@telecom-sudparis.eu>

Wana contribute ? Read the guide, send your contribution and get your name listed ;)

Table of Contents
=================

::

  0. What is it?
  1. Requirements
  2. Getting Ready
  3. Keystone 
  4. Glance
  5. Nova
  6. Cinder
  7. Adding a compute node
  8. Start your first VM
  9. Licencing
  10. Contacts
  11. References
  12. Credits
  13. To do

0. What is it?
==============

OpenStack Folsom Install Guide is an easy and tested way to create your own OpenStack plateform. 

Version 1.0

Status: testing 


1. Requirements
====================

:Node Role: NICs
:Control Node: em1 (10.111.80.201), em2.90 (10.222.90.201)
:Compute Node: em1 (10.111.80.202), em2.90 (10.222.90.202)

**Note 1:** This is my current network architecture, you can add as many compute node as you wish.

.. image:: http://i.imgur.com/RK6X7.jpg

2. Getting Ready
===============

2.1. Preparing Ubuntu 12.10
-----------------

* After you install Ubuntu 12.10 Server 64bits, Go to the sudo mode and don't leave it until the end of this guide::

   sudo su

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

2.2.Networking
------------
* First, take a good look at your working routing table::
   
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   0.0.0.0         10.222.90.254   0.0.0.0         UG    0      0        0 em2.90
   10.111.80.0     0.0.0.0         255.255.255.0   U     0      0        0 em1
   10.222.90.0     0.0.0.0         255.255.255.0   U     0      0        0 em2.90
 
* /etc/network/interfaces::

   auto lo
   iface lo inet loopback
 
   auto em1
   iface em1 inet static
   address 10.111.80.201
   netmask 255.255.255.0
  
   auto em2.90
   iface em2.90 inet static
   address 10.222.90.201
   netmask 255.255.255.0
   gateway 10.222.90.254
   dns-nameservers 8.8.8.8 8.8.4.4
   dns-search despexds.net
   vlan-raw-device em2

2.3. MySQL & RabbitMQ
------------

* Install MySQL::

   apt-get install mysql-server python-mysqldb

* Configure mysql to accept all incoming requests::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* Install RabbitMQ::

   apt-get install rabbitmq-server 

2.4. Node synchronization
------------------

* Install other services::

   apt-get install ntp

* Configure the NTP server to synchronize between your compute nodes and the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.5. Others
-------------------
* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf 

* Add 8021q to /etc/modules::

   echo "8021q" >> /etc/modules


3. Keystone
=====================================================================

This is how we install OpenStack's identity service:

* Start by the keystone packages::

   apt-get install keystone

* Create a new MySQL database for keystone::

   mysql -u root -p
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'localhost' IDENTIFIED BY 'keystonePass';
   quit;

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   connection = mysql://keystoneUser:keystonePass@localhost/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/danielitus/OpenStack-Folsom-Install-guide/tree/VLAN/2NICs/Keystone_Scripts>`_ of this git repository. Beware that you MUST comment every part related to Quantum if you don't intend to install it otherwise you will have trouble with your dashboard later::

   #Modify the HOST_IP variable before executing the scripts

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* Create a simple credential file and load it so you won't be bothered later::

   vi creds
   #Paste the following:
   export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=admin_pass
   export OS_AUTH_URL="http://10.111.80.201:5000/v2.0/"
   export OS_NO_CACHE=1
   # Load it:
   source creds

* To test Keystone, we use a simple curl request::

   curl http://10.111.80.201:35357/v2.0/endpoints -H 'x-auth-token: ADMIN'

* Reboot, test connectivity and check Keystone again.

4. Glance
=====================================================================

* After installing Keystone, we continue with installing image storage service a.k.a Glance::

   apt-get install glance

* Create a new MySQL database for Glance::

   mysql -u root -p
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'localhost' IDENTIFIED BY 'glancePass';
   quit;

* Update /etc/glance/glance-api-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.80.201
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update the /etc/glance/glance-registry-paste.ini with::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.80.201
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = service_pass

* Update /etc/glance/glance-api.conf with::

   sql_connection = mysql://glanceUser:glancePass@localhost/glance

* And::

   [paste_deploy]
   flavor = keystone

* Update the /etc/glance/glance-registry.conf with::

   sql_connection = mysql://glanceUser:glancePass@localhost/glance

* And::

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services::

   service glance-api restart; service glance-registry restart

* Synchronize the glance database::

   glance-manage db_sync

* To test Glance's well installation, we upload a new image to the store. Start by downloading the cirros cloud image to your node and then uploading it to Glance::

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* Now list the images to see what you have just uploaded::

   glance image-list

5. Nova
=================

* Start by adding this script to /etc/network/if-pre-up.d/iptablesload to forward traffic to em2.90::

   #!/bin/sh
   iptables -t nat -A POSTROUTING -o em2.90 -j MASQUERADE
   exit 0

* Install these packages::

   apt-get install nova-api nova-cert nova-doc nova-scheduler nova-consoleauth

* Prepare a Mysql database for Nova::

   mysql -u root -p
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
   quit;

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 10.111.80.201
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = service_pass
   signing_dirname = /tmp/keystone-signing-nova


* Change your /etc/nova/nova.conf to look like this::

    [DEFAULT]
    
    # LOGS/STATE
    verbose=True
    logdir=/var/log/nova
    state_path=/var/lib/nova
    lock_path=/run/lock/nova
    
    # AUTHENTICATION
    auth_strategy=keystone
    
    # SCHEDULER
    scheduler_driver=nova.scheduler.multi.MultiScheduler
    compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    
    # CINDER
    volume_api_class=nova.volume.cinder.API
    
    # DATABASE
    sql_connection=mysql://novaUser:novaPass@10.111.80.201/nova
    
    # COMPUTE
    libvirt_type=kvm
    libvirt_use_virtio_for_bridges=True
    start_guests_on_host_boot=True
    resume_guests_state_on_host_boot=True
    api_paste_config=/etc/nova/api-paste.ini
    allow_admin_api=True
    use_deprecated_auth=False
    nova_url=http://10.111.80.201:8774/v1.1/
    root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
    
    # APIS
    ec2_host=10.111.80.201
    ec2_url=http://10.111.80.201:8773/services/Cloud
    keystone_ec2_url=http://10.111.80.201:5000/v2.0/ec2tokens
    s3_host=10.111.80.201
    cc_host=10.111.80.201
    metadata_host=10.111.80.201
    #metadata_listen=0.0.0.0
    enabled_apis=ec2,osapi_compute,metadata
    
    # RABBITMQ
    rabbit_host=10.111.80.201
    
    # GLANCE
    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=10.111.80.201:9292
    
    # NETWORK
    network_manager=nova.network.manager.FlatDHCPManager
    force_dhcp_release=True
    dhcpbridge_flagfile=/etc/nova/nova.conf
    dhcpbridge=/usr/bin/nova-dhcpbridge
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    public_interface=em2
    flat_interface=em1
    flat_network_bridge=br100
    fixed_range=192.168.6.0/24
    network_size=256
    flat_network_dhcp_start=192.168.6.0
    flat_injected=False
    connection_type=libvirt
    multi_host=True

* Don't forget to update the ownership rights of the nova directory::

   chown -R nova. /etc/nova
   chmod 644 /etc/nova/nova.conf

* Add this line to the sudoers file::

   sudo visudo
   #Paste this line anywhere you like:
   nova ALL=(ALL) NOPASSWD:ALL

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

* Use the following command to create fixed network::
   
   nova-manage network create private --fixed_range_v4=192.168.6.0/24 --num_networks=1 --bridge=br100 --bridge_interface=em1 --network_size=256 --multi_host=T

* Create the floating IPs::

   nova-manage floating create --ip_range=10.222.90.128/26

* Create the floating to the nova project, run the next command many times as your network IPs::

    nova floating-ip-create

* Add ICMP ping and SSH access to the default security group::

    nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    nova secgroup-add-rule default tcp 22 22 0.0.0.0/0

6. Cinder
=================

Although Cinder is a replacement of the old nova-volume service, its installation is now a seperated from the nova install process.

* Install the required packages::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* Configure the iscsi services::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* Restart the services::
   
   service iscsitarget start
   service open-iscsi start

* Prepare a Mysql database for Cinder::

   mysql -u root -p
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'localhost' IDENTIFIED BY 'cinderPass';
   quit;

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 10.111.80.201
   service_port = 5000
   auth_host = 10.111.80.201
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = service_pass

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@localhost/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   #osapi_volume_listen_port=5900

* Then, synchronize your database::

   cinder-manage db sync

* Finally, don't forget to create a volumegroup and name it cinder-volumes::

   dd if=/dev/zero of=cinder-volumes bs=1 count=0 seek=2G
   losetup /dev/loop2 cinder-volumes
   fdisk /dev/loop2
   #Type in the followings:
   n
   p
   1
   ENTER
   ENTER
   t
   8e
   w

* Proceed to create the physical volume then the volume group::

   pvcreate /dev/loop2
   vgcreate cinder-volumes /dev/loop2

**Note:** Beware that this volume group gets lost after a system reboot. (Click `Here <https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/blob/master/Tricks%26Ideas/load_volume_group_after_system_reboot.rst>`_ to know how to load it after a reboot) 

* Restart the cinder services::

   service cinder-volume restart
   service cinder-api restart

7. Adding a compute node
=========================

7.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the controller node::
   
   sed -i 's/server ntp.ubuntu.com/server 10.111.80.201/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

* Add this script to /etc/network/if-pre-up.d/iptablesload to forward traffic to em2.90::

   #!/bin/sh
   iptables -t nat -A POSTROUTING -o em2.90 -j MASQUERADE
   exit 0

7.2.Networking
------------

* It's recommended to have two NICs but only one (em2.90) needs to be internet connected::
   
   auto em1
   iface em1 inet static
   address 10.111.80.202
   netmask 255.255.255.0
   dns-nameservers 8.8.8.8

   auto em2.90
   iface em2.90 inet static
   address 10.222.90.202
   netmask 255.255.255.0
   gateway 10.222.90.254

7.3 KVM
------------------

* KVM is needed as the hypervisor that will be used to create virtual machines.

* Enable live migration by updating /etc/libvirt/libvirtd.conf file::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* Edit libvirtd_opts variable in /etc/init/libvirt-bin.conf file::

   env libvirtd_opts="-d -l"

* Edit /etc/default/libvirt-bin file ::

   libvirtd_opts="-d -l"

* Restart the libvirt service to load the new values::

   service libvirt-bin restart

7.4. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install nova-compute nova-network nova-api-metadata

* Modify the /etc/nova/nova.conf like this::

    [DEFAULT]
    
    # LOGS/STATE
    verbose=True
    logdir=/var/log/nova
    state_path=/var/lib/nova
    lock_path=/run/lock/nova
    
    # AUTHENTICATION
    auth_strategy=keystone
    
    # SCHEDULER
    scheduler_driver=nova.scheduler.multi.MultiScheduler
    compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    
    # CINDER
    volume_api_class=nova.volume.cinder.API
    
    # DATABASE
    sql_connection=mysql://novaUser:novaPass@10.111.80.201/nova
    
    # COMPUTE
    libvirt_type=kvm
    libvirt_use_virtio_for_bridges=True
    start_guests_on_host_boot=True
    resume_guests_state_on_host_boot=True
    api_paste_config=/etc/nova/api-paste.ini
    allow_admin_api=True
    use_deprecated_auth=False
    nova_url=http://10.111.80.201:8774/v1.1/
    root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
    
    # APIS
    ec2_host=10.111.80.201
    ec2_url=http://10.111.80.201:8773/services/Cloud
    keystone_ec2_url=http://10.111.80.201:5000/v2.0/ec2tokens
    s3_host=10.111.80.201
    cc_host=10.111.80.201
    metadata_host=10.111.80.203
    
    # RABBITMQ
    rabbit_host=10.111.80.201
    
    # GLANCE
    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=10.111.80.201:9292
    
    # NETWORK
    network_manager=nova.network.manager.FlatDHCPManager
    force_dhcp_release=True
    dhcpbridge_flagfile=/etc/nova/nova.conf
    dhcpbridge=/usr/bin/nova-dhcpbridge
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    public_interface=em2.90
    flat_interface=em1
    flat_network_bridge=br100
    fixed_range=192.168.6.0/24
    network_size=256
    flat_network_dhcp_start=192.168.6.0
    flat_injected=False
    connection_type=libvirt
    multi_host=True

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

8. Your First VM
============

To start your first VM:

* Run a glance image-list first to find the UUID from the image to boot::

    nova boot --image deb3fd68-ff77-4994-881b-361d4142639e --flavor m1.tiny test

I Hope you enjoyed this guide, please if you have any feedbacks, don't hesitate.

9. Licensing
============

OpenStack Folsom Install Guide by Bilel Msekni is licensed under a Creative Commons Attribution 3.0 Unported License.

.. image:: http://i.imgur.com/4XWrp.png
To view a copy of this license, visit [ http://creativecommons.org/licenses/by/3.0/deed.en_US ].

10. Contacts
===========

Bilel Msekni: bilel.msekni@telecom-sudparis.eu

11. References
=================

* Configuring Floating IP addresses [http://www.mirantis.com/blog/configuring-floating-ip-addresses-networking-openstack-public-private-clouds/]
* Enabling Ping and SSH on VMs [http://docs.openstack.org/trunk/openstack-compute/admin/content/enabling-ping-and-ssh-on-vms.html]
* Instances not Receiving DHCP Offers with Nova-Network Method [https://github.com/mseknibilel/OpenStack-Folsom-Install-guide/issues/14]

12. Credits
=================

This work has been based on:

* Emilien Macchi's Folsom guide [https://github.com/EmilienM/openstack-folsom-guide]
* OpenStack Documentation [http://docs.openstack.org/trunk/openstack-compute/install/apt/content/]
* OpenStack Quantum Install [http://docs.openstack.org/trunk/openstack-network/admin/content/ch_install.html]
* Vlan configuration [https://wiki.ubuntu.com/vlan]

13. To do
=======

This guide is under testing. Your suggestions are always welcomed.
