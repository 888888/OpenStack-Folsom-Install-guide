==========================================================
  OpenStack Folsom 双网卡快速安装指南
==========================================================

关键字: 多节点安装，双网卡，Multi node OpenStack, Folsom, Quantum, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM, Ubuntu Server 12.10 (64 bits).

作者
==========

梁小白 <11315889@qq.com>


目录
=================

::

  0. 前言
  1. 需求说明
  2. 控制节点
  3. 网络节点
  4. 计算节点
  5. 启动一个虚拟机

0. 前言
==============

Openstack Folsom 发布好久了，但由于新的组件Quantum的加入，以及知识的跨度，比如同时需要系统管理及网络工程方面的知识，所以Folsom的安装还是挺费事的。
经过几天的测试，参考各种文档，终于完成了Folsom基于双网卡的安装，总结至此。


1. 需求说明
====================

:节点名称: NICs
:控制节点: eth0 (100.10.10.51), eth1 (192.168.100.51)
:网络节点: eth0 (100.10.10.52), eth2 (0.0.0.0)
:计算节点: eth0 (100.10.10.53)

**备注 1: ** 本文为双网卡安装Folsom设计,根据官方说明，网络节点最好采用三块网卡控制节点可以和计算节点合二为一.

**备注 2：** 本文安装指南环境为实现Folsom功能评估，力求简单方便，安全性差，不可用于生产环境。

**备注 3: ** 本文不适用于虚拟机环境.请使用物理计算机安装.

.. image:: http://i.imgur.com/4D51h.jpg

2. 控制节点
===============

2.1. 准备系统
-----------------
* 安装系统注意事项::

    - ubuntu-12.10-server-amd64.iso
    - 为Cinder服务预留独立分区 例如: /dev/sda5
    - 提前定义好各服务器主机名及ＩＰ，尽量别改，一定要改，请修改/etc/hosts中的对应关系

* 以下所有命令均在root权限下完成，所以在装好ubuntu后，请切换到root::

   sudo passwd
   su

* 更新系统(依据笔者经验，安装完Folsom环境后最好别再使用dist-upgrade,以免产生些许小问题，如虚拟机获得不了ip等..)::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade
   
* 因为要更新和下载的软件比较多，可以在空闲时间一次更新系统并提前安装所需要软件，以后只需要配置就行了:
   
   apt-get update && apt-get dist-upgrade -y && apt-get update -y && apt-get dist-upgrade -y && apt-get install -y rabbitmq-server ntp vlan bridge-utils keystone curl openssl glance quantum-server quantum-plugin-openvswitch nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms openstack-dashboard memcached python-mysqldb mysql-server 

2.2.配置网卡
------------
 
* 主控应该有一个外网网卡::

   #访问Openstack ＡＰＩ 
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

   #管理网络和虚拟机网络合二为一
   auto eth0
   iface eth0 inet static
   address 100.10.10.51
   netmask 255.255.255.0

* 重启网络服务::
   
   service networking restart

2.3. MySQL & RabbitMQ
------------

* 安装 MySQL 和 RabbitMQ::

   apt-get install mysql-server python-mysqldb rabbitmq-server 

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

   net.ipv4.conf.all.rp_filter = 0
   net.ipv4.conf.default.rp_filter = 0 
   sysctl net.ipv4.ip_forward=1
   
   # 检查一下
   sysctl -p 

2.6. Keystone
-------------------


* 安装组件::

   apt-get install keystone


* 编辑 /etc/keystone/keystone.conf 数据库连接::

   connection = mysql://root:password@100.10.10.51/keystone

* 重启keystone并初始化数据库::

   service keystone restart
   keystone-manage db_sync

* 使用 `自动化脚本 <https://github.com/888888/OpenStack-Folsom-Install-guide/tree/GRE/2NICs/Keystone_Scripts>`_ 创建keystone用户、服务、服务端点。为了简化，这里只创建admin一个用户，请不要修改此用户密码。 

    bash keystone_basic.sh
    执行一次，否则会创建多个service
    bash keystone_endpoints_basic.sh

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
* 安装组件

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

   wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
   glance image-create --name myFirstImage --is-public true --container-format bare \
    --disk-format qcow2 < cirros-0.3.0-x86_64-disk.img

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
   
    Binary           Host             Zone             Status     State Updated_At
    nova-cert        sm1u07           nova             enabled    :-)   2013-03-15 12:08:31
    nova-consoleauth sm1u07           nova             enabled    :-)   2013-03-15 12:08:30
    nova-scheduler   sm1u07           nova             enabled    :-)   2013-03-15 12:08:30
   

2.10. Cinder
-------------------

* 安装组件::

   apt-get install cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms

* 打开iscsi服务::

   sed -i 's/false/true/g' /etc/default/iscsitarget
   service iscsitarget start
   service open-iscsi start

* 修改 /etc/cinder/api-paste.ini 认证信息::

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

   service cinder-volume restart
   service cinder-api restart

2.11. 控制面板
-------------------

* 安装组件 ::

   apt-get install openstack-dashboard memcached


* dashboard依赖apache和memcache::

   service apache2 restart; service memcached restart

现在可以访问Dashboard了 **http://192.168.100.51/horizon** 用户名密码 **admin:password**.


3. 网络节点
=========================

3.1. 准备系统
------------------

* 安装ubuntu12.01::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade
   
    快速：
    apt-get update && apt-get dist-upgrade -y && apt-get install -y ntp vlan bridge-utils openvswitch-switch openvswitch-datapath-dkms quantum-plugin-openvswitch-agent quantum-dhcp-agent quantum-l3-agent
   

* 安装配置基本服务ntp,vlan,bridge-utils::

   apt-get install ntp vlan bridge-utils
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* 允许ip转发::

   vi /etc/sysctl.conf
   net.ipv4.conf.all.rp_filter = 0
   net.ipv4.conf.default.rp_filter = 0
   sysctl net.ipv4.ip_forward=1

3.2.配置网卡
------------

* 网络节点eth1网卡将做为虚拟机与互联网通讯端口，设置网卡为 promisc mode::
   
   #虚拟机外网出口
   auto eth1
   iface eth1 inet manual
   up ifconfig $IFACE 0.0.0.0 up
   up ip link set $IFACE promisc on
   down ip link set $IFACE promisc off
   down ifconfig $IFACE down

   #管理网络及内部通信
   auto eth0
   iface eth0 inet static
   address 100.10.10.52
   netmask 255.255.255.0

3.3. OpenVSwitch
------------------

* 安装虚拟交换机::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 创建网桥::

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


* 安装quantum组件::

   apt-get -y install quantum-dhcp-agent quantum-l3-agent quantum-plugin-openvswitch-agent

* 编辑/etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini :: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://root:password@100.10.10.51/quantum

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 100.10.10.52
   enable_tunneling = True

* 更新 /etc/quantum/l3_agent.ini::

   auth_url = http://100.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password
   metadata_ip = 100.10.10.51
   metadata_port = 8775
   use_namespaces = False

* 修改 /etc/quantum/dhcp_agent.ini::

   use_namespaces = False

* 修改/etc/quantum/quantum.conf ::
  
   rabbit_host = 100.10.10.51

* 重启所有服务::

   service quantum-dhcp-agent restart
   service quantum-l3-agent restart
   service quantum-plugin-openvswitch-agent restart

4. 计算节点
=========================

4.1. 准备系统
------------------

* 更新升级::

   apt-get update
   apt-get upgrade
   apt-get dist-upgrade

   快速：
    apt-get update && apt-get dist-upgrade -y && apt-get install -y ntp vlan bridge-utils cpu-checker kvm libvirt-bin pm-utils openvswitch-switch openvswitch-datapath-dkms  quantum-plugin-openvswitch-agent nova-compute-kvm

* 安装 ntp vlan bridge-utils::

   apt-get install ntp vlan bridge-utils
   sed -i 's/server ntp.ubuntu.com/server 100.10.10.51/g' /etc/ntp.conf
   service ntp restart  

* 允许ＩＰ转发::

   vi /etc/sysctl.conf
   
   net.ipv4.conf.all.rp_filter = 0
   net.ipv4.conf.default.rp_filter = 0
   sysctl net.ipv4.ip_forward=1

4.2.配置网卡
------------

* vi /etc/network/interfaces ::
   
   # 管理网络和内部通讯网络
   auto eth0
   iface eth0 inet static
   address 100.10.10.53
   netmask 255.255.255.0

4.3 KVM
------------------

* 确认硬件支持虚拟化::

   apt-get install cpu-checker
   kvm-ok

* 安装kvm组件::

   apt-get install -y kvm libvirt-bin pm-utils

* 编辑libvirt设备列表支持tun  /etc/libvirt/qemu.conf::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* 删除kvm默认网络配置 ::

   virsh net-destroy default
   virsh net-undefine default

* 允许动态迁移 ::

   vi /etc/libvirt/libvirtd.conf 
   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"
   
   vi /etc/init/libvirt-bin.conf
   env libvirtd_opts="-d -l"

   vi /etc/default/libvirt-bin
   libvirtd_opts="-d -l"

   service libvirt-bin restart

4.4. OpenVSwitch
------------------

* 安装 openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* 创建网桥 bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

4.5. Quantum
------------------

* 安装 Quantum openvswitch agent::

   apt-get -y install quantum-plugin-openvswitch-agent

* 编辑OVS配置 /etc/quantum/plugins/openvswitch/ovs_quantum_plugin.ini:: 

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

* 修改rabbitMQ IP ::

   vi /etc/quantum/quantum.conf
   rabbit_host = 100.10.10.51

* 重启所有服务::

   service quantum-plugin-openvswitch-agent restart

4.6. Nova
------------------

* 安装nova compute组件::

   apt-get install nova-compute-kvm

* 修改 /etc/nova/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystone.middleware.auth_token:filter_factory
   auth_host = 100.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = admin
   admin_user = admin
   admin_password = password
   signing_dirname = /tmp/keystone-signing-nova

* 编辑 /etc/nova/nova-compute.conf::
   
   [DEFAULT]
   libvirt_type=kvm
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* 修改 /etc/nova/nova.conf  ::

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

* 使用 <http://192.168.10.51/horizon> 管理虚拟机
* 编辑安全组，允许所有协议,tcp,udp,icmp

    root@sm1u07:~# nova secgroup-list-rules default
    Please enter password for encrypted keyring: 
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| icmp        | -1        | 255     | 0.0.0.0/0 |              |
| tcp         | 1         | 65535   | 0.0.0.0/0 |              |
| udp         | 1         | 65535   | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+

* 使用脚本 `quantum.sh <https://raw.github.com/888888/OpenStack-Folsom-Install-guide/GRE/2NICs/Keystone_Scripts/quantum.sh>`_ 为admin创建相关的网络，即虚拟机内网和外网
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
    
* 目前horizon不支持quantum的floatingip操作,通过quantum 命令行为vm 分配floatingip,
    
     quantum floatingip-create --port_id <port_id> <ext_net_id>
  
* 大功告成，现在你可以去dashboard中用vnc登录vm，测试一下各个网络是否通畅
