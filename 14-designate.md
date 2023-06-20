# Designate Install

> ![Designate logo](/images/designate.png)

## 1. CONTROLLER NODE

### Database setup

```bash
mysql
```

```bash
CREATE DATABASE designate;
```

```bash
GRANT ALL PRIVILEGES ON designate.* TO 'designate'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON designate.* TO 'designate'@'%' IDENTIFIED BY 'password123';
exit
```

### Set up openstack components

1. Source admin credentials

```
source ~/.adminrc
```

2. Create **designate** user

```
openstack user create --domain default --password password123 designate
```

3. Grant admin role to **designate** user

```
openstack role add --project service --user designate admin
```

4. Create service entry

```
openstack service create --name designate --description "DNS" dns
```

5. Create API endpoints

```bash
for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  dns $i http://controller:9001; done
```

### Install packages

```bash
apt-get install designate-api designate-central -y
```

### Configure Designate

1. Backup and sanitize **/etc/designate/designate.conf**

```bash
cp -p /etc/designate/designate.conf /etc/designate/designate.conf.bak
grep -Ev '^(#|$)' /etc/designate/designate.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/designate/designate.conf
```

2. Modify **/etc/designate/designate.conf**

```yaml
[DEFAULT]
transport_url = rabbit://openstack:password123@controller:5672/

[database]
connection = mysql+pymysql://designate:password123@controller/designate

[storage:sqlalchemy]
connection = mysql+pymysql://designate:password123@controller/designate

[service:api]
listen = 0.0.0.0:9001
auth_strategy = keystone
enable_api_v2 = True
enable_api_admin = True
enable_host_header = True
enabled_extensions_admin = quotas, reports

[keystone_authtoken]
auth_type = password
username = designate
password = password123
project_name = service
project_domain_name = Default
user_domain_name = Default
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211

[service:worker]
enabled = True
notify = True
```

### Populate database

```bash
su -s /bin/sh -c "designate-manage database sync" designate
```

### Start services

```
systemctl start designate-central designate-api

systemctl enable designate-central designate-api
```

## 2. NETWORK NODE

1. Install packages

```
apt install designate-worker designate-producer designate-mdns -y
```

### config files

1. Backup and sanitize **/etc/designate/designate.conf**

```bash
cp -p /etc/designate/designate.conf /etc/designate/designate.conf.bak
grep -Ev '^(#|$)' /etc/designate/designate.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/designate/designate.conf
```

2. Modify **/etc/designate/designate.conf**

```yaml
[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller:5672/

[database]
connection = mysql+pymysql://designate:password123@controller/designate

[storage:sqlalchemy]
connection = mysql+pymysql://designate:password123@controller/designate

[service:api]
listen = 0.0.0.0:9001
auth_strategy = keystone
enable_api_v2 = True
enable_api_admin = True
enable_host_header = True
enabled_extensions_admin = quotas, reports

[keystone_authtoken]
auth_type = password
username = designate
password = password123
project_name = service
project_domain_name = Default
user_domain_name = Default
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211

[service:worker]
enabled = True
notify = True
```

### Install BIND packages

```
apt install bind9 bind9utils -y
```

### Configure BIND

1. Create RNDC key

```
rndc-confgen -a -k designate -c /etc/bind/rndc.key
chown bind:designate /etc/bind/rndc.key
chmod 640 /etc/bind/rndc.key
```

2. Backup **/etc/bind/named.conf.options**

```bash
cp -p /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
```

3. Edit **/etc/bind/named.conf.options**

* test query ip range here. may need to allow mgmt network

```yaml
include "/etc/bind/rndc.key";

options {
    #...
    allow-new-zones yes;
    request-ixfr no;
    listen-on port 53 { any; };
    listen-on-ipv6 port 53 { none; };
    recursion no;
    allow-query { any; };
};

controls {
  inet 0.0.0.0 port 953
    allow { localhost; } keys { "designate"; };
};
```

3. Restart bind

```
systemctl restart bind9.service
```

### Configure pools

1. Create pool file **/etc/designate/pools.yaml**

```yaml
- name: default
  # The name is immutable. There will be no option to change the name after
  # creation and the only way will to change it will be to delete it
  # (and all zones associated with it) and recreate it.
  description: Default Pool

  attributes: {}

  # List out the NS records for zones hosted within this pool
  # This should be a record that is created outside of designate, that
  # points to the public IP of the controller node.
  ns_records:
    - hostname: <network node hostname>.
      priority: 1

  # List out the nameservers for this pool. These are the actual BIND servers.
  # We use these to verify changes have propagated to all nameservers.
  nameservers:
    - host: <ip of bind server>
      port: 53

  # List out the targets for this pool. For BIND there will be one
  # entry for each BIND server, as we have to run rndc command on each server
  targets:
    - type: bind9
      description: BIND9 Server 1

      # List out the designate-mdns servers from which BIND servers should
      # request zone transfers (AXFRs) from.
      # This should be the IP of the controller node.
      # If you have multiple controllers you can add multiple masters
      # by running designate-mdns on them, and adding them here.
      masters:
        - host: <ip of mdns host>
          port: 5354

      # BIND Configuration options
      options:
        host: <ip of mdns host>
        port: 53
        rndc_host: <ip of mdns host>
        rndc_port: 953
        rndc_key_file: /etc/bind/rndc.key
```

2. Set permissions on pools file

```
chmod 640 /etc/designate/pools.yaml
chgrp designate /etc/designate/pools.yaml
```

2.  Update pools

```
su -s /bin/sh -c "designate-manage pool update" designate
```

### Finalize install

1. Start services

```
systemctl start designate-worker designate-producer designate-mdns

systemctl enable designate-worker designate-producer designate-mdns
```

3. Verify services

```bash
ps -aux|grep designate
```

## 3. CONTROL NODE

### Install Horizon dashboard for designate

1. Install packages

```
apt install python3-designate-dashboard -y
```

2. Restart apache

```
service apache2 restart
```

### Verify ops

1. Show service list

```
openstack dns service list
```

2. Source demo credentials

```bash
source ~/.demorc
```

3. Create a zone

```
openstack zone create --email dnsmaster@example.com example.com.

openstack zone create --email dnsmaster@example.com 0.0.10.in-addr.arpa.

openstack zone list
```

4. Create a recordset

```
openstack recordset create --record '10.0.0.10' --type A example.com. node1
openstack recordset create --record 'node01.example.com.' --type PTR 0.0.10.in-addr.arpa. 10

openstack recordset list example.com.

openstack recordset list 0.0.10.in-addr.arpa.
```

5. Delete zone

```
openstack zone delete example.com.
```
