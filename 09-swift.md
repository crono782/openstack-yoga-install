# Install swift

> ![Swift logo](/images/swift.png)

## 1. CONTROLLER NODE

### Create Openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **swift** user and add role:

```bash
openstack user create --domain default --password password123 swift

openstack role add --project service --user swift admin
```

* Create **swift** service:

```bash
openstack service create --name swift \
  --description "Openstack Object Storage" object-store
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  object-store $i http://controller:8080/v1/AUTH_%\(project_id\)s; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install swift swift-proxy python3-swiftclient \
  python3-keystoneclient python3-keystonemiddleware \
  memcached -y
```

2. Create **/etc/swift** dir:

```bash
install -d /etc/swift -o root -g swift
```

3. Get copy of **proxy-server.conf**:

```bash
curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/proxy-server.conf-sample

chgrp swift /etc/swift/proxy-server.conf
```

4. Backup an sanitize **/etc/swift/proxy-server.conf**:

```bash
cp -p /etc/swift/proxy-server.conf /etc/swift/proxy-server.conf.bak
grep -Ev '^(#|$)' /etc/swift/proxy-server.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/swift/proxy-server.conf
```

3. Edit **/etc/swift/proxy-server.conf** sections:

```yaml
[DEFAULT]
#...
bind_port = 8080
user = swift
swift_dir = /etc/swift

[pipeline:main]
# remove tempurl and tempauth, add authtoken and keystoneauth
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
#...
account_autocreate = True

[filter:keystoneauth]
use = egg:swift#keystoneauth
#...
operator_roles = admin,member

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = password123
delay_auth_decision = True

[filter:cache]
use = egg:swift#memcache
#...
memcache_servers = controller:11211
```

## 1. OBJECT STORAGE NODE

### Utility setup

1. Install packages:

```bash
apt-get install xfsprogs rsync -y
```

2. Format extra disks

```
mkfs.xfs /dev/vdc

mkdir -p /srv/node/vdc
```

3. get block id of new vol and add to fstab, then mount

```
blkid
```

```yaml
UUID="<UUID>" /srv/node/vdc xfs noatime 0 2
```

```
mount /srv/node/vdc
```

4. create or modify **/etc/rsyncd.conf**:

```yaml
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_INTERFACE_IP_ADDRESS

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock
```

5. enable rsync in **/etc/default/rsync**

```yaml
RSYNC_ENABLE=true
```

6. start rsync:

```
service rsync start
```

### Install componenets

1. install packages

```
apt-get install swift swift-account swift-container swift-object -y
```

2. pull sample config files

```
curl -o /etc/swift/account-server.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/account-server.conf-sample
curl -o /etc/swift/container-server.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/container-server.conf-sample
curl -o /etc/swift/object-server.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/object-server.conf-sample
```

3. Backup an sanitize **/etc/swift/account-server.conf**:

```bash
cp -p /etc/swift/account-server.conf /etc/swift/account-server.conf.bak
grep -Ev '^(#|$)' /etc/swift/account-server.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/swift/account-server.conf
```

4. edit **/etc/swift/account-server.conf**

```yaml
[DEFAULT]
#...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6202
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon account-server

[filter:recon]
use = egg:swift#recon
#...
recon_cache_path = /var/cache/swift
```

5. Backup an sanitize **/etc/swift/container-server.conf**:

```bash
cp -p /etc/swift/container-server.conf /etc/swift/container-server.conf.bak
grep -Ev '^(#|$)' /etc/swift/container-server.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/swift/container-server.conf
```

6. edit **/etc/swift/container-server.conf**

```yaml
[DEFAULT]
#...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6201
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon container-server

[filter:recon]
use = egg:swift#recon
#...
recon_cache_path = /var/cache/swift
```

7. Backup an sanitize **/etc/swift/object-server.conf**:

```bash
cp -p /etc/swift/object-server.conf /etc/swift/object-server.conf.bak
grep -Ev '^(#|$)' /etc/swift/object-server.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/swift/object-server.conf
```

8. edit **/etc/swift/object-server.conf**

```yaml
[DEFAULT]
#...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6200
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon object-server

[filter:recon]
use = egg:swift#recon
#...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
```

9. set perms on **/srv/node**

```
chown -R swift:swift /srv/node
```

10. create cache dir and set perms

```
mkdir -p /var/cache/swift
chown -R root:swift /var/cache/swift
chmod -R 775 /var/cache/swift
```

## CONTROLLER NODE

### Create initial rings and distribute

1. cd to **/etc/swift** dir

2. create rings

```
swift-ring-builder account.builder create 8 1 1
swift-ring-builder container.builder create 8 1 1
swift-ring-builder object.builder create 8 1 1
```

3. add storage devices to rings

```
swift-ring-builder account.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6202 --device DEVICE_NAME --weight DEVICE_WEIGHT

swift-ring-builder container.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6201 --device DEVICE_NAME --weight DEVICE_WEIGHT

swift-ring-builder object.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6200 --device DEVICE_NAME --weight DEVICE_WEIGHT
```

4. verify rings

```
swift-ring-builder account.builder
swift-ring-builder container.builder
swift-ring-builder object.builder
```

5. rebalance rings

```
swift-ring-builder account.builder rebalance
swift-ring-builder container.builder rebalance
swift-ring-builder object.builder rebalance
```

6. distribute rings to object storage nodes

```
scp /etc/swift/*.gz STORAGE_NODE:/etc/swift
```

### Configure swift

1. get sample conf file

```
curl -o /etc/swift/swift.conf https://opendev.org/openstack/swift/raw/branch/stable/wallaby/etc/swift.conf-sample
```

2. Backup an sanitize **/etc/swift/swift.conf**:

```bash
cp -p /etc/swift/swift.conf /etc/swift/swift.conf.bak
grep -Ev '^(#|$)' /etc/swift/swift.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/swift/swift.conf
```

3. modify file **/etc/swift/swift.conf**

```yaml
[swift-hash]
#...
swift_hash_path_suffix = suffix_4332469b98761c60e0c2e3e40daeefe9
swift_hash_path_prefix = prefix_952b70ca75466fe48354359f958b2a42

[storage-policy:0]
#...
name = Policy-0
default = yes
```

4. distribute conf file to **/etc/swift** on storage nodes

```
scp /etc/swift/swift.conf STORAGE_NODE:/etc/swift
```

5. ensure proper ownership of files on controll and all storage nodes

```
chown -R root:swift /etc/swift
```

6. restart services

```
service memcached restart
service swift-proxy restart
```

7. start all swift services

```
systemctl stop swift-proxy

swift-init all start
```

### verify ops

1. get creds

```
source ~/.adminrc
```

2. check swift stats

```
swift stat
```

3. create container, object, test, etc

```
openstack container create container1

echo 'sample text' > test.txt

openstack object create container1 test.txt

rm -f test.txt

openstack object list container1

openstack object save container1 test.txt

openstack object delete container1 test.txt
```


