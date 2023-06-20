# Install Cinder

> ![Cinder logo](/images/cinder.png)

## 1: CONTROLLER NODE

* Access database as root:

```bash
mysql
```

* Create **cinder** database:

```sql
CREATE DATABASE cinder;
```

* Grant proper access to **cinder** user:

```sql
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' identified by 'password123';
exit
```

* Source .adminrc

```bash
source .adminrc
```

* Create **cinder** user and add role:

```bash
openstack user create --domain default --password password123 cinder

openstack role add --project service --user cinder admin
```

* Create **cinder v2** services:

```bash
openstack service create --name cinderv2 \
  --description "Openstack Block Storage" volumev2

openstack service create --name cinderv3 \
  --description "Openstack Block Storage" volumev3
```

* Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  volumev2 $i http://controller:8776/v2/%\(project_id\)s; \
  done

for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  volumev3 $i http://controller:8776/v3/%\(project_id\)s; \
  done
```

* Install packages:

```bash
apt install cinder-api cinder-scheduler -y
```

* Backup an sanitize **/etc/cinder/cinder.conf**:

```bash
cp -p /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
grep -Ev '^(#|$)' /etc/cinder/cinder.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/cinder/cinder.conf
```

* Edit **/etc/cinder/cinder.conf** sections:

```yaml
[database]
# ...
connection = mysql+pymysql://cinder:password123@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone
my_ip = 10.10.10.11

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password123

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

* Fix packaging bug

```bash
install -d /var/lib/cinder/tmp -o cinder -g cinder
```

* Populate database

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

* Configure compute to use cinder. Edit **/etc/nova/nova.conf**:

```yaml
[cinder]
os_region_name = RegionOne
```

* Finalize install

```bash
service nova-api restart
systemctl enable cinder-scheduler
service cinder-scheduler restart
service apache2 restart
```

## 2. STORAGE NODE

* Install packages:

```bash
apt install lvm2 thin-provisioning-tools -y
```

```bash
apt install cinder-volume -y
```

* Create LVM volumes. Use appropriate block device name instead of /dev/vdb

```bash
pvcreate /dev/vdb

vgcreate cinder-volumes /dev/vdb
```

* Create LVM filter

```yaml
devices {
...
filter = [ "a/sdb/", "r/.*/" ]
```

* If LVM used on OS, include those vols. Do same on compute nodes:

```yaml
filter = [ "a/sda/", "a/sdb/", "r/.*/" ]
```

* Backup an sanitize **/etc/cinder/cinder.conf**:

```bash
cp -p /etc/cinder/cinder.conf /etc/cinder/cinder.conf.bak
grep -Ev '^(#|$)' /etc/cinder/cinder.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/cinder/cinder.conf
```

* Edit **/etc/cinder/cinder.conf** sections:

```yaml
[database]
# ...
connection = mysql+pymysql://cinder:password123@controller/cinder

[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone
my_ip = 10.10.10.14
enabled_backends = lvm
glance_api_servers = http://controller:9292

# comment these out... will add in lvm section
#iscsi_helper = lioadm
#volume_name_template = volume-%s
#volume_group = cinder-volumes

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = password123

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
volume_name_template = volume-%s
target_protocol = iscsi
target_helper = lioadm

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

* Finalize Install

```bash
systemctl enable cinder-volume
service cinder-volume restart
```

* Verify Ops

```bash
source .adminrc

openstack volume service list

openstack volume create --size 1 test-vol

openstack volume list

openstack volume delete test-vol
```

### Configure Backup Service

* Do this AFTER swift install

1. Install packages

```bash
apt install cinder-backup -y
```

2. Get swift url

```bash
openstack catalog show object-store
```

3. Edit **/etc/cinder/cinder.conf**:

```yaml
[DEFAULT]
# ...
backup_driver = cinder.backup.drivers.swift.SwiftBackupDriver
backup_swift_url = SWIFT_URL
swift_catalog_info = object-store:swift:publicURL

# OR WAY SIMPLER TO USE FILE BASED BACKUPS FOR TESTING

backup_driver = cinder.backup.drivers.posix.PosixBackupDriver
```

4. Finalize install

```bash
systemctl enable cinder-backup
service cinder-backup restart
```

### Enable backups in horizon

1. Modify settings in **/etc/openstack-dashboard/local_settings.py**:

```yaml

OPENSTACK_CINDER_FEATURES = {
  "enable_backup": True
}
```

```bash
service apache2 restart
```

5. Verify ops

```bash
sourc ~/.adminrc

openstack volume create --size 1 bu-test-vol

openstack volume backup create --name bu-test-vol-backup bu-test-vol

openstack volume backup list

openstack container list

openstack volume backup delete bu-test-vol-backup

openstack volume delete bu-test-vol
```