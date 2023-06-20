# Nova Install

> ![Nova logo](/images/nova.png)

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **nova_api**, **nova**, and **nova_cell0** databases:

```sql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
```

3. Grant proper access to **nova** user:

```sql
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' identified by 'password123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' identified by 'password123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' identified by 'password123';
exit
```

### Create openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **nova** user and add role:

```bash
openstack user create --domain default --password password123 nova

openstack role add --project service --user nova admin
```

* Create **nova** service:

```bash
openstack service create --name nova \
  --description "Openstack Compute" compute
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  compute $i http://controller:8774/v2.1; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install nova-api nova-conductor nova-novncproxy nova-scheduler -y
```

2. Backup an sanitize **/etc/nova/nova.conf**:

```bash
cp -p /etc/nova/nova.conf /etc/nova/nova.conf.bak
grep -Ev '^(#|$)' /etc/nova/nova.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/nova/nova.conf
```

2. Edit **/etc/nova/nova.conf** sections:

```yaml
[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller:5672/
my_ip = 10.10.10.11

[api]
# ...
auth_strategy = keystone

[api_database]
# ...
# remove other connections
connection = mysql+pymysql://nova:password123@controller/nova_api

[database]
# ...
# remove other conections
connection = mysql+pymysql://nova:password123@controller/nova

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = password123

[oslo_concurrency]
# ...
# ensure this dir exists
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = password123

[vnc]
# ...
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

3. Populate database:

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
```

4. Register cell0 database:

```bash
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

5. Create cell1 cell:

```bash
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

6. Populate nova database:

```bash
su -s /bin/sh -c "nova-manage db sync" nova
```

7. Verify registration:

```bash
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

8. Restart compute services:

```bash
service nova-api restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
```

## 2. COMPUTE NODE

### Install and configure componenets

1. Install packages:

```bash
apt install nova-compute -y
```

2. Backup an sanitize **/etc/nova/nova.conf**:

```bash
cp -p /etc/nova/nova.conf /etc/nova/nova.conf.bak
grep -Ev '^(#|$)' /etc/nova/nova.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/nova/nova.conf
```

3. Edit **/etc/nova/nova.conf** sections:

```yaml
[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
my_ip = 10.10.10.12

[api]
# ...
auth_strategy = keystone

[glance]
# ...
api_servers = http://controller:9292

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = password123

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = password123

[vnc]
# ...
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

4. Determine kvm support

```bash
grep -Ec '(vmx|svm)' /proc/cpuinfo
```

* *ONLY if no kvm support configure libvirt to use qemu*:

```yaml
[libvirt]
# ...
virt_type = qemu
```

5. Remove unsused sqlite db

```
rm -f /var/lib/nova/nova.sqlite
```

6. Restart compute service:

```bash
service nova-compute restart
```

## 3. CONTROLLER NODE

### Add compute node to cell database

1. Source .adminrc

```bash
source .adminrc
```

2. Discover compute hosts:

```bash
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

### Verify ops

```bash
openstack compute service list

openstack catalog list

openstack image list

nova-status upgrade check
```