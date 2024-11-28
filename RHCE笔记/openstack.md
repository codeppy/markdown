## openstack

### 一、openstack组件

1、Horizon:Dashboard

2、nova

Controller节点：提供各种组件api的接口，如nova-api等

compute节点：

3、glance

镜像服务

4、swift对象存储

5、cinder块存储

6、neutron

7、heat编排服务

8、ceilmeter计量服务,监控资源

9、keystore

### 二、部署工具（方式）

1、packstack

2、devstack

3、ansible

==4、开源方式部署==

5、TripleO Openstack on Openstack

首先搭建一套精简版 openstack，然后all-in-one heat 自动化编排工具 参数填写 进行部署

如：华为 HCS deploy-HCS 部署工具批量部署

### 三、搭建开源U版 openstack

#### 1、实验环境

3台controller节点（8g4vcpus100G）+2台compute节点(4g4vcpus100G),开启VT-X虚拟化

每台主机两块网卡，一个外网，一个内部通讯

#### 2、网络规划，静态ip地址及主机名设置

```bash
controller 192.168.1.188
compute01 192.168.1.189
compute02 192.168.1.190

hostnamectl set-hostname "controller"
```

#### 3、关闭防火墙和selinux并设置开机不自启，还有交换分区

```bash
systemctl disable firewalld.service --now
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#关闭swap分区
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### 4、设置hosts，配置免密登录

```bash
vim /etc/hosts
192.168.1.188 controller
192.168.1.189 compute01
192.168.1.190 compute02

ssh-keygen
ssh-copy-id -i root@主机IP /root/.ssh/id_rsa.pub
```

#### 5、环境检查

如果你是英文环境此步**跳过**。
如果不是英文环境：
**Controller & Computel 都要执行**

```bash
vim /etc/environment

LANG=en_US.utf-8
LC_ALL=en_US.utf-8

reboot #重启系统
```

#### 6、检查yum源，是否正常

yum install list

#### 7、同步系统时钟：

**Controller & Computel 都要执行**
安装chrony工具
yum -y install chrony
vim /etc/chrony.conf
添加
server Controller iburst

systemctl restart chronyd.service
systemctl enable chronyd.service --now

#### 8、 检查系统时区：

Controller & Computel 都要执行
执行
timedatectl | grep “Time zone”
返回结果如果时区不是“Asia/Shanghai”

进行时区设置：
timedatectl set-timezone Asia/Shanghai

#### 9、更新系统，安装U版repo源(仅U版使用命令如下)

-----所有节点都要装哈

```bash
cd /etc/yum.repos.d
yum -y install centos-release-openstack-ussuri
yum config-manager --set-enabled powertools
yum -y upgrade
reboot #更新完重启下系统
yum -y install python3-openstackclient
yum -y install openstack-selinux
```

#### 10、安装数据库(仅控制节点)

```bash
yum -y install mariadb mariadb-server python2-PyMySQL
vim /etc/my.cnf.d/openstack.cnf #添加内容如下
[mysqld]
bind-address = controller

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8

systemctl enable mariadb.service --now #启动数据库
mysql_secure_installation #加固数据库的
systemctl restart mariadb.service
```

#### 11、安装消息队列MQ(仅控制节点)

```bash
yum install rabbitmq-server
systemctl enable rabbitmq-server.service --now
rabbitmqctl add_user openstack redhat #创建一个openstack用户，密码为redhat
rabbitmqctl set_permissions openstack ".*" ".*" ".*" #允许用户有配置、写、读的权限
rabbitmqctl list_users #查询用户列表
systemctl restart rabbitmq-server.service
```

#### 12、安装memcache(仅控制节点)

```bash
yum -y install memcached python3-memcached
vim /etc/sysconfig/memcached #配置监听
OPTIONS="-l 127.0.0.1,::1,controller"

systemctl enable memcached.service --now
```

13、安装ETCD数据库

```bash
yum -y install etcd
vim /etc/etcd/etcd.conf #修改地址为控制节点的管理地址（此处只能填写ip，域名不可以）
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://10.0.0.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_NAME="controller"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER_STATE="new"

systemctl enable etcd --now
```

### 四、安装和配置keystone(控制节点)

```bash
$ mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS'; #修改密码
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS'; #修改密码
exit
yum -y install openstack-keystone httpd python3-mod_wsgi
vim /etc/keystone/keystone.conf
[database]
# ...
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone #修改密码

[token]
# ...
provider = fernet
```

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone #生成表结构

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap --bootstrap-password ADMIN_PASS \ #密码要修改
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
  
vim /etc/httpd/conf/httpd.conf
ServerName controller:80

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
systemctl enable httpd.service --now

vim admin.openrc
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS #修改密码
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

### 五、安装glance组件(控制节点)

```bash
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject
openstack user create --domain default --password-prompt myuser
openstack role create myrole
openstack role add --project myproject --user myuser myrole
```

```bash
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
. admin-openrc
openstack user create --domain default --password-prompt glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
yum -y install openstack-glance
vim /etc/glance/glance-api.conf #编辑里面内容并注意修改密码

[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

[keystone_authtoken]
# ...
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api.service --now

#glance服务验证测试
. admin-openrc
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

glance image-create --name "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility=public
  
glance image-list
```

### 六、安装和部署placement(控制节点)

```bash
vim /etc/httpd/conf.d/00-placement-api.conf #添加内容如图
mysql -u root -p
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS'; #修改密码
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS'; #修改密码
exit
. admin-openrc
openstack user create --domain default --password-prompt placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
yum -y install openstack-placement-api
vim /etc/placement/placement.conf
[placement_database]
# ...
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS

su -s /bin/sh -c "placement-manage db sync" placement
systemctl restart httpd

#组件安装验证
. admin-openrc
placement-status upgrade check
pip3 install osc-placement
vim /etc/httpd/conf.d/00-placement-api.conf #添加内容如下
<Directory /usr/bin>
      <IfVersion >= 2.4>
          Require all granted
      </IfVersion>
      <IfVersion < 2.4>
          Order allow,deny
          Allow from all
      </IfVersion>
  </Directory>
systemctl restart httpd
```

添加位置如下cat /etc/httpd/conf.d/00-placement-api.conf

![image-20240601181515807](./openstack.assets/image-20240601181515807.png)

```bash
#必须添加上图内容，便可拥有权限查询
openstack --os-placement-api-version 1.2 resource class list --sort-column name
openstack --os-placement-api-version 1.6 trait list --sort-column name
```

### 七、nova安装部署（分控制节点和计算节点）

##### 控制节点安装

```

```



### 八、neutron安装部署



```bash
net.bridge.bridge-nf-call-iptables
net.bridge.bridge-nf-call-ip6tables
```

如果执行提示找不到命令，在所有节点上安装bridge-utils即可

```bash
yum -y install bridge-utils
modprobe br_netfilter
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
sysctl -a|grep bridge
```



### 九、dashboard安装



编辑以下文件，找到WEBROOT = ‘/’ 修改为WEBROOT = ‘/dashboard’

[root@controller ~]#vim /usr/share/openstack-dashboard/openstack_dashboard/defaults.py

[root@controller~]#vim /usr/share/openstack-dashboard/openstack_dashboard/test/settings.py

重启memecache和httpd服务

[root@controller ~]#systemctl restart httpd.service memcached.service



### 十、各组件详解

keystone

#多域支持 +133

keystone几种令牌方式：UUID、fernet、pki



role针对不同用户、租户以及不同组件权限控制

/etc/openstack-dashboard/

![image-20240729222231680](./openstack.assets/image-20240729222231680.png)
