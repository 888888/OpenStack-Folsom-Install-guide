==========================================================
  OpenStack Folsom 双网卡快速安装指南
==========================================================

Keywords: 多节点安装，双网卡，Multi node OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.10 (64 bits).

Authors
==========

梁小白 <11315889@qq.com>


Table of Contents
=================

::

  0. What is it?
  1. 需求说明
  2. 控制节点
  3. Network Node
  4. Compute Node
  5. Start your first VM
  6. Licencing
  7. Contacts
  8. Acknowledgement
  9. Credits
  10. To do

0. What is it?
==============

OpenStack Folsom Install Guide is an easy and tested way to create your own OpenStack plateform. 

Version 3.1

Status: Testing


1. 需求说明
====================

:Node Role: NICs
:Control Node: eth0 (100.10.10.51), eth1 (192.168.100.51)
:Network Node: eth0 (100.10.10.52), eth2 (0.0.0.0)
:Compute Node: eth0 (100.10.10.53)

**备注 1: ** 本文为双网卡安装Folsom设计,根据官方说明，网络节点最好采用三块网卡控制节点可以和计算节点合二为一.
**备注 2：** 本文安装指南环境为实现Folsom功能评估，力求简单方便，安全性差，不可用于生产环境。
**备注 3: ** 本文不适用于虚拟机环境.请使用物理计算机安装.

.. image:: http://i.imgur.com/4D51h.jpg

2. 控制节点
===============

2.1. 准备 Ubuntu 12.10
-----------------
* 控制节点同时承载Cinder服务，安装时请单独创建一个lvm分区给cinder使用::

  例如: /dev/sda5

* 以下所有命令均在root权限下完成，所以在装好ubuntu后，请切换到root::
   启用root用户
   
   sudo passwd
   输入新的root密码

   sudo su

* 更新系统(依据笔者经验，安装完Folsom环境后最好别再使用dist-upgrade,以免产生些许小问题，如虚拟机获得不了ip等..)::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade
   
   **备注 : ** 因为要更新和下载的软件比较多，可以在空闲时间一次安装:
   
   apt-get update && apt-get dist-upgrade -y && apt-get update -y && apt-get dist-upgrade -y && apt-get install -y rabbitmq-server ntp vlan bridge-utils keystone curl openssl glance quantum-server quantum-plugin-openvswitch nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms openstack-dashboard memcached python-mysqldb mysql-server 

2.2.配置网卡
------------
 
* Only one NIC on the controller should be internet connected::

   #Exposes OpenStack API to the internet 
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

   #Management & network configuration
   auto eth0
   iface eth0 inet static
   address 100.10.10.51
   netmask 255.255.255.0

* 重启 networking services::
   
   service networking restart

2.3. MySQL & RabbitMQ
------------

* Install MySQL::

   apt-get install mysql-server python-mysqldb

* 配置Mysql监听所有地址::

   sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
   service mysql restart

* 为了简化安装，以后所有连接mysql服务均使用 root:password登录,将root权限更改为所有主机可以访问(默认只能本机访问)
  
  mysql -uroot -ppassword
  
  use mysql;
  update user set host='%' where user='root' and host='localhost';
  flush privileges;
 
* 创建所有必须的数据库::
  
  create database keystone;
  create database nova;
  create database glance;
  create database cinder;
  create database quantum;
   
* Install RabbitMQ::

   apt-get install rabbitmq-server 

2.4. 节点时间同步
------------------

* 安装时间服务器,其它节点时间同此服务器同步::

   apt-get install ntp

   sed -i 's/server ntp.ubuntu.com/server ntp.ubuntu.com\nserver 127.127.1.0\nfudge 127.127.1.0 stratum 10/g' /etc/ntp.conf
   service ntp restart  

2.5. Others
-------------------
* 安装其它服务::

   apt-get install vlan bridge-utils

* 允许IP转发::

   vi /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1
   检查一下
   sysctl -p 

2.6. Keystone
-------------------

This is how we install OpenStack's identity service:

* Start by the keystone packages::

   apt-get install keystone


* 编辑 /etc/keystone/keystone.conf 数据库连接::

   connection = mysql://root:password@100.10.10.51/keystone

* 重启keystone并初始化数据库::

   service keystone restart
   keystone-manage db_sync

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/888888/OpenStack-Folsom-Install-guide/tree/master/Keystone_Scripts>`_ of this git repository. Beware that you MUST comment every part related to Quantum if you don't intend to install it otherwise you will have trouble with your dashboard later::

   #Modify the HOST_IP and HOST_IP_EXT variable before executing the scripts

   chmod +x keystone_basic.sh
   chmod +x keystone_endpoints_basic.sh

   ./keystone_basic.sh
    执行一次，否则会创建多个service
   ./keystone_endpoints_basic.sh

* 创建/root/novarc文件，写入以下内容::

    export OS_TENANT_NAME=admin
    export OS_TENANT_ID=c7fb80d964a24ab1bc0fd370696c804e
    export OS_USERNAME=admin
    export OS_PASSWORD=password
    export OS_AUTH_URL="http://127.0.0.1:35357/v2.0"
    export OS_REGION_NAME=RegionOne
    export OS_IDENTITY_API_VERSION=2.0
    export SERVICE_TOKEN=ADMIN
    export SERVICE_ENDPOINT="http://127.0.0.1:35357/v2.0"
    
    各项值请根据实际情况替换
    keystone tenant-list
    将获取的tenant_id替换到novarc
    
    source /root/novarc
    echo "source /root/novarc" >> ~/.bashrc


2.7. Glance
-------------------

   apt-get install glance


* 分别修改 /etc/glance/glance-api-paste.ini 和/etc/glance/glance-registry-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password

* 分别修改 /etc/glance/glance-api.conf 和/etc/glance/glance-registry.conf ::

   sql_connection = mysql://root:password@100.10.10.51/glance

   [paste_deploy]
   flavor = keystone


* 重启glance服务并同步glance数据库::

   service glance-api restart; service glance-registry restart

   glance-manage db_sync
   
* 测试glance 服务，不输出任何结果代表成功::

    glance index
    
* 上传个镜像::

   mkdir images
   cd images
   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

* 再查看一下::

   glance image-list

2.8. Quantum
-------------------

* 安装组件::

   apt-get install quantum-server quantum-plugin-openvswitch


* 修改 /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini,移动后文件最后，有模板:: 

   [DATABASE]
   sql_connection = mysql://root:password@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True

* 修改 /etc/quantum/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password

* 重启 quantum server::

   service quantum-server restart

2.9. Nova
-------------------

* 安装组件::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy


* 修改 /etc/nova/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password
   signing_dirname = /tmp/keystone-signing-nova

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=100.10.10.51
   ec2_host=100.10.10.51
   ec2_dmz_host=100.10.10.51
   rabbit_host=100.10.10.51
   cc_host=100.10.10.51
   dmz_cidr=169.254.169.254/32
   metadata_host=100.10.10.51
   metadata_listen=0.0.0.0
   nova_url=http://100.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@100.10.10.51/nova
   ec2_url=http://100.10.10.51:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://100.10.10.51:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=100.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=192.168.100.51
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://100.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=admin
   quantum_admin_username=admin
   quantum_admin_password=password
   quantum_admin_auth_url=http://100.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* 初始化nova数据库::

   nova-manage db sync

* 重启所有nova服务::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* 检查nova服务，有笑脸图标，证明服务正常::

   nova-manage service list
   
    Binary           Host                                 Zone             Status     State Updated_At
    nova-cert        sm1u07                               nova             enabled    :-)   2013-03-15 12:08:31
    nova-consoleauth sm1u07                               nova             enabled    :-)   2013-03-15 12:08:30
    nova-scheduler   sm1u07                               nova             enabled    :-)   2013-03-15 12:08:30
   

2.10. Cinder
-------------------

* 安装组件::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* Configure the iscsi services::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* Restart the services::
   
   service iscsitarget start
   service open-iscsi start

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 100.10.10.51
   service_port = 5000
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = password
   admin_password = password

* 修改the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://root:password@100.10.10.51/cinder
   api_paste_confg = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone

* 初始化cinder数据库::

   cinder-manage db sync


* 创建cinder使用的物理卷及卷组::

   pvcreate /dev/sda5
   vgcreate cinder-volumes /dev/sda5


* Restart the cinder services::

   service cinder-volume restart
   service cinder-api restart

2.11. Horizon
-------------------

* To install horizon, proceed like this ::

   apt-get install openstack-dashboard memcached


* Reload Apache and memcached::

   service apache2 restart; service memcached restart

现在可以访问Dashboard了 **192.168.100.51/horizon** with credentials **admin:password**.


3. Network node
=========================

3.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade
   
   快速：apt-get update && apt-get dist-upgrade -y && apt-get install -y ntp vlan bridge-utils openvswitch-switch openvswitch-datapath-dkms quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the 控制节点::
   
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

3.2.Networking
------------

* 网络节点eth1网卡将做为虚拟机与互联网通讯端口，设置网卡为 promisc mode::
   
   auto eth1
   iface eth1 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

   auto eth0
   iface eth0 inet static
   address 100.10.10.52
   netmask 255.255.255.0

3.3. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int is used for VM integration	
   ovs-vsctl add-br br-int

   #br-ex is used for accessing internet.
   ovs-vsctl add-br br-ex
   ovs-vsctl br-set-external-id br-ex bridge-id br-ex
   ovs-vsctl add-port br-ex eth1
   启动br-ex
   ip link set br-ex up

3.4. Quantum
------------------

We need to install the l3 agent, dhcp agent and the openVSwitch plugin agent

* Install quantum DHCP and l3 agents::

   apt-get -y install quantum-dhcp-agent quantum-l3-agent quantum-plugin-openvswitch-agent

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://quantumUser:quantumPass@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 100.10.10.52
   enable_tunneling = True

* In addition, update the /etc/quantum/l3_agent.ini::

   auth_url = http://100.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password
   metadata_ip = 100.10.10.51
   metadata_port = 8775
   use_namespaces = False

* Edit /etc/quantum/dhcp_agent.ini::

   use_namespaces = False

* 修改稿rabbitMQ IP ::
  
   vi /etc/quantum/quantum.conf
   rabbit_host = 100.10.10.51


* Restart all the services::

   service quantum-dhcp-agent restart
   service quantum-l3-agent restart
   service quantum-plugin-openvswitch-agent restart

4. Compute Node
=========================

4.1. Preparing the Node
------------------

* Update your system::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

   快速：apt-get update && apt-get dist-upgrade -y && apt-get install -y ntp vlan bridge-utils cpu-checker kvm libvirt-bin pm-utils openvswitch-switch openvswitch-datapath-dkms quantum-plugin-openvswitch-agent nova-compute-kvm

* Install ntp service::

   apt-get install ntp

* Configure the NTP server to follow the 控制节点::
   
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* Install other services::

   apt-get install vlan bridge-utils

* Enable IP_Forwarding::

   nano /etc/sysctl.conf
   # Uncomment net.ipv4.ip_forward=1, to save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

4.2.Networking
------------

* Perform the following::
   
   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 100.10.10.53
   netmask 255.255.255.0

4.3 KVM
------------------

* make sure that your hardware enables virtualization::

   apt-get install cpu-checker
   kvm-ok

* Normally you would get a good response. Now, move to install kvm and configure it::

   apt-get install -y kvm libvirt-bin pm-utils

* Edit the cgroup_device_acl array in the /etc/libvirt/qemu.conf file to::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* Delete default virtual bridge ::

   virsh net-destroy default
   virsh net-undefine default

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

4.4. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

4.5. Quantum
------------------

* Install the Quantum openvswitch agent::

   apt-get -y install quantum-plugin-openvswitch-agent

* Edit the OVS plugin configuration file /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://root:password@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 100.10.10.53
   enable_tunneling = True

* Make sure that your rabbitMQ IP in /etc/quantum/quantum.conf is set to the 控制节点::
   
   rabbit_host = 100.10.10.51

* Restart all the services::

   service quantum-plugin-openvswitch-agent restart

4.6. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install nova-compute-kvm

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password
   signing_dirname = /tmp/keystone-signing-nova

* Edit /etc/nova/nova-compute.conf file ::
   
   [DEFAULT]
   libvirt_type=kvm
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT]
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   scheduler_driver=nova.scheduler.simple.SimpleScheduler
   s3_host=100.10.10.51
   ec2_host=100.10.10.51
   ec2_dmz_host=100.10.10.51
   rabbit_host=100.10.10.51
   cc_host=100.10.10.51
   dmz_cidr=169.254.169.254/32
   metadata_host=100.10.10.51
   metadata_listen=0.0.0.0
   nova_url=http://100.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@100.10.10.51/nova
   ec2_url=http://100.10.10.51:8773/services/Cloud 
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone
   keystone_ec2_url=http://100.10.10.51:5000/v2.0/ec2tokens
   # Imaging service
   glance_api_servers=100.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=100.10.10.53
   vncserver_listen=0.0.0.0 

   # Network settings
   network_api_class=nova.network.quantumv2.api.API
   quantum_url=http://100.10.10.51:9696
   quantum_auth_strategy=keystone
   quantum_admin_tenant_name=admin
   quantum_admin_username=admin
   quantum_admin_password=password
   quantum_admin_auth_url=http://100.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   # Compute #
   compute_driver=libvirt.LibvirtDriver

   # Cinder #
   volume_api_class=nova.volume.cinder.API
   osapi_volume_listen_port=5900

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done   

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

5. 创建虚拟机
============

* 使用http://192.168.10.51/horizon管理虚拟机
* 编辑安全组，允许所有协议,tcp,udp,icmp
* 使用脚本quantum.sh 为admin创建相关的网络，即虚拟机内网和外网
* 查看创建好的网络

  root@hp4u:~# quantum net-list
+--------------------------------------+-----------+--------------------------------------+
| id                                   | name      | subnets                              |
+--------------------------------------+-----------+--------------------------------------+
| 14dbb282-c74a-4784-bfc3-351f7ca3d034 | ext_net   | 95bddb90-84dc-4579-99b8-798a393a3edf |
| d402e168-cbda-4345-8ffa-015e6a1c4aa1 | admin-net | 8ef3c4dd-a265-421c-afa2-6cff28ae2c74 |
+--------------------------------------+-----------+--------------------------------------+
root@hp4u:~# quantum router-list
+--------------------------------------+-----------------+--------------------------------------------------------+
| id                                   | name            | external_gateway_info                                  |
+--------------------------------------+-----------------+--------------------------------------------------------+
| 623b68f4-967a-4028-9a92-dc5a7d3e16e8 | provider-router | {"network_id": "14dbb282-c74a-4784-bfc3-351f7ca3d034"} |
+--------------------------------------+-----------------+--------------------------------------------------------+

* 修改 /etc/quantum/l3_agent.ini :

    gateway_external_network_id = 14dbb282-c74a-4784-bfc3-351f7ca3d034
    router_id = 623b68f4-967a-4028-9a92-dc5a7d3e16e8
    
    service quantum-l3-agent restart  
* 使用控制面板创建一个虚拟机，并记录vm-uuid，勇冠vm-uuid获取vm的端口id

    quantum port-list -- --device_id <vm-uuid>
    
* 通过quantum 命令行为vm 分配floatingip,

  quantum floatingip-create --port_id <port_id> <ext_net_id>
  **备注：** 目前horizon不支持quantum的floatingip操作


