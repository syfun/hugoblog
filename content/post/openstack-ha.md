+++
title = "Openstack HA环境搭建"
thumbnail = ""
tags = []
categories = ["云计算"]
date = "2015-11-25T19:01:12+08:00"
description = ""
draft = true
+++


本文主要是Openstack HA环境搭建的全纪录。先说下机器配置：

HOST |　eth0 | eth1 | OS | Role
---- |  --- | ----- | --- 
controller1 | 10.0.0.11 |   | ubuntu 14.04.3 | MASTER
controller2 | 10.0.0.12 |  |  ubuntu 14.04.3 | BACKUP
dashboard   | 10.0.0.10 | 192.168.1.10 | ubuntu 14.04.3 |

VIP是10.0.0.20。这里要说明一下，我们从本地只能登录到dashboard上，然后通过dashboard登陆到其他两台机器。dashboard上只装horizon服务。

Openstack版本是liberty。

<!--more-->

每台机器的/etc/hosts文件都需要添加:

```
10.0.0.10     dashboard
10.0.0.11     controller1
10.0.0.12     controller2
10.0.0.20     controller
```


然后，HA是用Keepalived+haproxy实现。

## Keepalived 和 haproxy

首先，我们安装服务。ubuntu系统直接使用apt-get。

```
apt-get install keepalived haproxy
```

#### 配置keepalived
首先是controller1的keepalived配置：

/etc/keepalived/keepalived.conf
```ini
MASTER:
global_defs {
    router_id controller1
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    ##use_vmac vrrp.250
    priority 101
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    track_script {
        check_haproxy
    }
    virtual_ipaddress {
        10.0.0.20/24
    }
}
```

check_haproxy.sh用来检测当haproxy挂掉时，停掉本机的keepalived服务，VIP漂移到controller2。

/etc/keepalived/check_haproxy.sh
```sh
#!/usr/bin/env bash

res=`ps -C haproxy --no-heading | wc -l`
if [ $res -eq 0 ]; then
    service keepalived stop
fi
```

controller2的keepalived配置：

/etc/keepalived/keepalived.conf
```ini
global_defs {
    router_id controller2
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    #use_vmac vrrp.250
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    track_script {
        chceck_haproxy
    }
    virtual_ipaddress {
        10.0.0.20/24
    }
}
```

同样需要chceck_haproxy.sh脚本。

编辑/etc/default/keepalived，修改DAEMON_ARGS。

```
DAEMON_ARGS="-d -D -S 0"
```

在/etc/rsyslog.d/下添加keepalived.conf，内容只有一条记录。

```
local0.*        /var/log/keepalived.log
```

#### 配置haproxy

两台controller上配置相同。/etc/haproxy/haproxy.cfg如下：

```ini
global
          log 127.0.0.1   local3
          chroot /var/lib/haproxy
          stats socket /run/haproxy/admin.sock mode 660 level admin
          stats timeout 30s
          user haproxy
          group haproxy
          daemon

          # Default SSL material locations
          ca-base /etc/ssl/certs
          crt-base /etc/ssl/private

          # Default ciphers to use on SSL-enabled listening sockets.
          # For more information, see ciphers(1SSL). This list is from:
          #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
          ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
          ssl-default-bind-options no-sslv3

  defaults
          log     global
          mode    http
          option  httplog
          option  dontlognull
          log 127.0.0.1 local3
          timeout connect 5000
          timeout client  50000
          timeout server  50000
          errorfile 400 /etc/haproxy/errors/400.http
          errorfile 403 /etc/haproxy/errors/403.http
          errorfile 408 /etc/haproxy/errors/408.http
          errorfile 500 /etc/haproxy/errors/500.http
          errorfile 502 /etc/haproxy/errors/502.http
          errorfile 503 /etc/haproxy/errors/503.http
          errorfile 504 /etc/haproxy/errors/504.http

listen glance_api_cluster
  bind *:9292
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server controller1 10.0.0.11:10003 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10003 check inter 2000 rise 2 fall 5

listen glance_registry_cluster
  bind *:9191
  balance  source
  option  tcpka
  option  tcplog
  server controller1 10.0.0.11:10004 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10004 check inter 2000 rise 2 fall 5

listen keystone_admin_cluster
  bind *:35357
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server controller1 10.0.0.11:10001 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10001 check inter 2000 rise 2 fall 5

listen keystone_public_internal_cluster
  bind *:5000
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server controller1 10.0.0.11:10002 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10002 check inter 2000 rise 2 fall 5

listen nova_compute_api_cluster
  bind *:8774
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server controller1 10.0.0.11:10005 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10005 check inter 2000 rise 2 fall 5


listen neutron_api_cluster
  bind *:9696
  balance  source
  option  tcpka
  option  httpchk
  option  tcplog
  server controller1 10.0.0.11:10006 check inter 2000 rise 2 fall 5
  server controller2 10.0.0.12:10006 check inter 2000 rise 2 fall 5
```

修改/etc/rsyslog.conf中配置，取消两条记录的注释。
```
#$ModLoad imudp  ==》$ModLoad imudp
#$UDPServerRun 514 ==》$UDPServerRun 514
```

/etc/rsyslog.d下面已经有了个49-haporxy.conf，添加

```
local3.*        /var/log/haproxy.log
```

重启rsyslog。

```
service rsyslog restart
```

启动keepalived和haroxy。

```sh
service haproxy restart
service keepalived restart
```

#### 测试VIP飘逸

在controller1上`ip -a`能在eth0上看到10.0.0.20，就说明配置成功。然后停掉haproxy，会触发关闭keepalived服务，VIP漂移到controller2上。这时在controller2的eth0上能看到10.0.0.20。

## 环境需求

#### NTP

在dashboard上安装NPT服务，其他两台直接同步dashboard。直接运行下面的命令，具体的我就不说了，因为我也不清楚，哈哈。

```sh
apt-get install ntp
sed -i 's/^server/#server/g' /etc/ntp.conf
sed -i 's/^restrict/#restrict/g' /etc/ntp.conf

cat <<something >> /etc/ntp.conf
SYNC_HWCLOCK=yes
server ntp.ubuntu.com
server 127.127.1.0
fudge 127.127.1.0 stratum 10
restrict 127.0.0.1
something

service ntp restart
hwclock -w
```

在controller1和controller2上运行下面的命令：

```sh
apt-get install ntp
sed -i  "/^server.*/d" /etc/ntp.conf

cat <<EOF >> /etc/ntp.conf
server dashboard
EOF

sed -i -e "s/restrict 127.0.0.1/restrict 0.0.0.0/g" /etc/ntp.conf

apt-get autoremove ntp
echo "*/5 * * * * root /usr/sbin/ntpdate dashboard;/sbin/hwclock -w" > /etc/cron.d/sync_time
```
不要问我为什么先装ntp后来又卸载掉，我也不知道，肯定没错就是了。

#### Openstack安装包

14.04的源里openstack的版本不是L版的，所以需要另外添加L版的源。

```sh
apt-get install software-properties-common
add-apt-repository cloud-archive:liberty
apt-get update && apt-get dist-upgrade
apt-get install python-openstackclient
```

#### 数据库

具体的以后添加，就当mysql的主机是SQLHOST。

#### 消息队列

具体的以后添加，就当rabbitmq的主机是MQHOST。

```sh
rabbitmqctl add_user openstack RABBIT_PASS
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

下面就开始正式安装了。建立数据库和同步数据库只需要在controller1上做，其他的controller2上也需要做。

## 安装Keystone

在安装服务前，我们先做一些额外的工作。

#### 建立数据库

登陆mysql建立keystone数据库，并进行授权。

```
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
```

####　禁止keytone自动启动

在L版中，我们在apache上运行keystone的wsgi服务。所以我们需要禁止keystone的自动启动。

```
echo "manual" > /etc/init/keystone.override
```

#### 安装服务并配置

安装keystone服务。

```
apt-get install keystone apache2 libapache2-mod-wsgi \
  memcached python-memcache
```

在下面这个配置中列出的只是需要修改或添加或者重要的地方，并不是只有这些项。后面如果没有特殊说明，都是如此。

```ini
[DEFAULT]
admin_token = ADMIN_TOKEN
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@SQLHOST/keystone

[memcache]
servers = localhost:11211

[token]
provider = uuid
driver = memcache

[revoke]
driver = sql
```

配置好后，就可以同步数据库了。

```sh
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

#### 配置apache

在/etc/apached2/apache2.conf中添加一行：

```
ServerName controller1
```

> controller2上填controller2

新建/etc/apache2/sites-available/wsgi-keystone.conf，写入配置:

```ini
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
```

接着做这个操作：

```
ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

重启apache2。

```sh
service apache2 restart
```

#### 建立endpoint

添加环境变量

```
export OS_TOKEN=ADMIN_TOKEN
export OS_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

创建keystone的服务和endpoint。

```sh
openstack service create \
  --name keystone --description "OpenStack Identity" identity

openstack endpoint create --region RegionOne \
  identity public http://controller:5000/v2.0

openstack endpoint create --region RegionOne \
  identity internal http://controller:5000/v2.0

openstack endpoint create --region RegionOne \
  identity admin http://controller:35357/v2.0
```

创建机构和用户。

```sh
openstack project create --domain default \
  --description "Service Project" service

openstack user create --domain default \
  --password AMIN_PASS admin

openstack role create admin

openstack role add --project admin --user admin admin

openstack project create --domain default \
  --description "Service Project" service
```

#### admin用户环境变量

创建admin-openrc.sh

```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

然后`source admin-openrc.sh`。

或者将其添加到.bashrc中。建议使用这种方式，下次登陆的时候不用再次source脚本就可以使用openstack命令

使用`openstack token issue`测试一下。

## 安装glance

先建立glance数据库。

```
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
```

创建glance认证。

```
openstack user create --domain default --password GLANCE_PASS glance

openstack role add --project service --user glance admin

openstack service create --name glance \
    --description "OpenStack Image service" image

openstack endpoint create --region RegionOne \
      image public http://controller:9292

openstack endpoint create --region RegionOne \
      image internal http://controller:9292

openstack endpoint create --region RegionOne \
      image admin http://controller:9292
```

安装服务。

```sh
apt-get install glance python-glanceclient python-swiftclient
```

修改/etc/glance/glance-api.conf。

```ini
[DEFAULT]
notification_driver = noop

[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@SQLHOST/glance

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone

[glance_store]
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

修改/etc/glance/glance-registry.conf。

```ini
[DEFAULT]
notification_driver = noop

[database]
connection = mysql+pymysql://glance:GLANCE_DBPASS@SQLHOT/glance

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
flavor = keystone
```

同步数据库。

```sh
su -s /bin/sh -c "glance-manage db_sync" glance
```

重启服务。

```sh
service glance-registry restart
service glance-api restart
```

可以上传一个镜像测试一下。

```sh
glance image-create --name "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility public --progress
```
