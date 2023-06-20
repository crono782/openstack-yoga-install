# Trove Install

> ![Trove logo](/images/trove.png)

* TODO: Split into multinode install

## 1. CONTROLLER NODE

## Database setup

1. Access database as root:

```bash
mysql
```

2. Create **trove** database:

```sql
CREATE DATABASE trove;
```

3. Grant proper access to **trove** user:

```sql
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'password123';
exit
```

### Create Openstack resources

1. Source .adminrc

```bash
source .adminrc
```

2. Create service

```bash
openstack user create --domain default --password password123 trove
```

3. Add admin role

```bash
openstack role add --project service --user trove admin
```

4. Create service

```bash
openstack service create --name trove --description "Database" database
```

5. Create api endpoints

```bash
for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  database $i http://controller:8779/v1.0/%\(tenant_id\)s; done
```

### Install and configure components

1. Install packages

```bash
apt-get install trove-api trove-taskmanager trove-conductor python3-troveclient -y
```

2. Backup and sanitize **/etc/trove/trove.conf**

```bash
cp -p /etc/trove/trove.conf /etc/trove/trove.conf.bak
grep -Ev '^(#|$)' /etc/trove/trove.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/trove/trove.conf
```

3. Edit **/etc/trove/trove.conf** sections

```yaml
[DEFAULT]
log_dir = /var/log/trove
transport_url = rabbit://openstack:password123@controller:5672
control_exchange = trove
trove_api_workers = 5
network_driver = trove.network.neutron.NeutronDriver
taskmanager_manager = trove.taskmanager.manager.Manager
default_datastore = mysql
cinder_volume_type = lvm-trove
reboot_time_out = 300
usage_timeout = 900
agent_call_high_timeout = 1200

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123

[service_credentials]
auth_url = http://controller:5000
region_name = RegionOne
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123

[database]
connection = mysql+pymysql://trove:password123@controller/trove

[mariadb]
tcp_ports = 3306,4444,4567,4568

[mysql]
tcp_ports = 3306

[postgresql]
tcp_ports = 5432

[redis]
tcp_ports = 6379,16379
```

4. Backup and sanitize **/etc/trove/trove-guestagent.conf***

```bash
cp -p /etc/trove/trove-guestagent.conf /etc/trove/trove-guestagent.conf.bak
grep -Ev '^(#|$)' /etc/trove/trove-guestagent.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/trove/trove-guestagent.conf
```

5. Edit **/etc/trove/trove-guestagent.conf** sections

```yaml
[DEFAULT]
log_file = trove-guestagent.log
log_dir = /var/log/trove/
ignore_users = os_admin
control_exchange = trove
transport_url = rabbit://openstack:password123@controller:5672/
command_process_timeout = 60
use_syslog = False

[service_credentials]
auth_url = http://controller:5000
region_name = RegionOne
project_domain_name = Default
user_domain_name = Default
project_name = service
username = trove
password = password123
```

### Populate database

```bash
su -s /bin/sh -c "trove-manage db_sync" trove
```

### Finalize install

```bash
service trove-api restart
service trove-taskmanager restart
service trove-conductor restart
```

### Obtain and upload trove image

1. Obtain image

```bash
wget https://tarballs.opendev.org/openstack/trove/images/trove-wallaby-guest-ubuntu-bionic.qcow2
```

* HOTFIX: Fix trove backups

Since the move to docker-based databases in a single image, an issue has arisen where trove backups will complete, however the status check is broken with the new implementation. Changes have not made it to stable branch as of this writing. Fixing this yourself requires some sort of code injection which can be done a number of ways. Here, will used guestfish to mount the image and inject the fix here. See the proposed fix: https://review.opendev.org/c/openstack/trove/+/807474

```bash
apt install libguestfs-tools -y

guestfish --rw -a trove-wallaby-guest-ubuntu-bionic.qcow2

run

mount /dev/sda1 /

edit /opt/guest-agent-venv/lib/python3.6/site-packages/trove/guestagent/datastore/service.py

# on or around line 491, remove:
# result = output[-1]

# on or around line 497, remove:
# backup_result = BACKUP_LOG_RE.match(result)
# and replace with (ensure good indentation)
            backup_result = None
            output.reverse()
            for result in output:
                backup_result = BACKUP_LOG_RE.match(result)
                if backup_result:
                    break

exit
```

2. Upload trove service image to glance

```bash
openstack image create trove-guest-ubuntu-bionic --file=trove-wallaby-guest-ubuntu-bionic.qcow2 --disk-format=qcow2 --container-format=bare --tag=trove --private

openstack image list
```

### Create dedicated trove volume type

```bash
openstack volume type create lvm-trove --private 
```

### Create mariadb datastore

```bash
openstack datastore version create 10.4 mariadb mariadb '' --image-tags trove --active
```

### Load validation rules

```bash
su -s /bin/sh -c "trove-manage db_load_datastore_config_parameters mariadb 10.4 /usr/lib/python3/dist-packages/trove/templates/mariadb/validation-rules.json"
```

### Create cloud init for mariadb

1. Create cloudinit directory

```
install -d /etc/trove/cloudinit -o trove -g trove
```

2. Create mariadb cloudinit file **/etc/trove/cloudinit/mariadb.cloudinit**

```yaml
#cloud-config
write_files:
  - path: /etc/trove/controller.conf
    content: |
      CONTROLLER=10.10.10.11
  - path: /etc/hosts
    content: |
      10.10.10.11 controller
    append: true
runcmd:
  - chmod 644 /etc/trove/controller.conf
```

3. Set permissions

```
chown -R trove:trove /etc/trove/cloudinit
```

### Create appropriate flavor to run db instances

```bash
openstack flavor create --disk 5 --ram 1024  --vcpus 1 db.m1.small
```

### Get info and create db instance adjust params to suit

```bash

openstack network list

openstack database instance create mariadb-103 \
  --flavor db.m1.small \
  --size 1 \
  --nic net-id=<NET ID OF DESIRED NETWORK> \
  --database mydb --users dqueen:password123 \
  --datastore mariadb --datastore-version 10.3 \
  --is-public \
  --allowed-cidr 10.10.10.0/24 --allowed-cidr 203.0.113.0/24
  ```

### Verify ops

```bash
openstack database instance list
 
mysql -h <floating ip> -u user -p
```

### NOTES
if need to debug, create keypair in service project, add "nova_keypair = <name> in trove.conf under [DEFAULT]. update security group rules on generated group in service project to allow ping/ssh. node cannot resolve controller so add to hosts file via cloud init... dont forget to allow traffic via iptables if needed

### TODOs

* investigate better tag based approach to image selection.
* maybe dedicated trove mgmt network and sec group for better handling.
* add postgresql and mysql