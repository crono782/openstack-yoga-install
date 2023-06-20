# Zun Install

> ![Zun logo](/images/zun.png)

# Install Docker Engine

## 1. COMPUTE NODE

### Uninstall old versions

```
apt-get remove docker docker-engine docker.io containerd runc
```

### Set up repo

1. Install packages

```
sudo apt-get install \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```

2. Add GPG Key

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. Add stable docker repo

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Install docker engine

```
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io -y
```

# Install Kuryr-libnetwork

## 1. CONTROLLER NODE

### Create Openstack resources

```bash
source .adminrc
```

```bash
openstack user create --domain default --password password123 kuryr
```

```bash
openstack role add --project service --user kuryr admin
```

## 2. COMPUTE NODE

### Create zun user

```
groupadd --system kuryr

useradd --home-dir "/var/lib/kuryr" \
--create-home \
--system \
--shell /bin/false \
-g kuryr \
kuryr

mkdir -p /etc/kuryr
chown kuryr:kuryr /etc/kuryr
```

### Set up packages

1. Clone and install kuryr-libnetwork code

```
apt-get install python3-pip -y

cd /var/lib/kuryr

git clone https://opendev.org/openstack/kuryr-libnetwork.git --branch stable/wallaby

chown -R kuryr:kuryr kuryr-libnetwork

cd kuryr-libnetwork

pip3 install -r requirements.txt

python3 setup.py install
```

2. Generate sample config files

```bash
su -s /bin/sh -c "./tools/generate_config_file_samples.sh" kuryr
su -s /bin/sh -c "cp etc/kuryr.conf.sample \
      /etc/kuryr/kuryr.conf" kuryr
```

### config files

1. Backup and sanitize **/etc/kuryr/kuryr.conf**

```bash
cp -p /etc/kuryr/kuryr.conf /etc/kuryr/kuryr.conf.bak
grep -Ev '^(#|$)' /etc/kuryr/kuryr.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/kuryr/kuryr.conf
```

2. Modify **/etc/kuryr/kuryr.conf**

```yaml
[DEFAULT]
#...
bindir = /usr/local/libexec/kuryr

[neutron]
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
username = kuryr
user_domain_name = default
password = password123
project_name = service
project_domain_name = default
auth_type = password
```

### Create systemd unit file

* **/etc/systemd/system/kuryr-libnetwork.service**

```yaml
[Unit]
Description = Kuryr-libnetwork - Docker network plugin for Neutron

[Service]
ExecStart = /usr/local/bin/kuryr-server --config-file /etc/kuryr/kuryr.conf
CapabilityBoundingSet = CAP_NET_ADMIN
AmbientCapabilities = CAP_NET_ADMIN

[Install]
WantedBy = multi-user.target
```

### Finalize install

```
systemctl enable kuryr-libnetwork
systemctl start kuryr-libnetwork

systemctl restart docker
```

## 3. CONTROLLER NODE

### Verify Ops

1. Source rc file

```bash
source ~/.adminrc
```

2. Create ipv4 network

```
docker network create --driver kuryr --ipam-driver kuryr \
      --subnet 10.10.0.0/16 --gateway=10.10.0.1 test_net

docker network ls

docker run --net test_net cirros ifconfig
```

## 1. CONTROLLER NODE

### Database setup

1. Access mysql database

```
mysql
```

2. Create **zun** database

```
CREATE DATABASE zun;
```

3. Grant access to **zun** user

```bash
GRANT ALL PRIVILEGES ON zun.* TO 'zun'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON zun.* TO 'zun'@'%' IDENTIFIED BY 'password123';
exit
```

### Create Openstack resources

```bash
source .adminrc
```

```bash
openstack user create --domain default --password password123 zun
```

```bash
openstack role add --project service --user zun admin
```

```bash
openstack service create --name zun --description "Container Service" container
```

```bash
for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  container $i http://controller:9517/v1; done
```

### Create zun user

```
groupadd --system zun

useradd --home-dir "/var/lib/zun" \
--create-home \
--system \
--shell /bin/false \
-g zun \
zun

mkdir -p /etc/zun
chown zun:zun /etc/zun
```

### Set up packages

1. Clone and install zun code

```
cd /var/lib/zun

git clone https://opendev.org/openstack/zun.git --branch stable/wallaby

chown -R zun:zun zun

cd zun

pip install -r requirements.txt

python3 setup.py install
```

2. Generate sample config file

```bash
su -s /bin/sh -c "oslo-config-generator \
    --config-file etc/zun/zun-config-generator.conf" zun

su -s /bin/sh -c "cp etc/zun/zun.conf.sample \
    /etc/zun/zun.conf" zun
```

3. Copy api-paste.ini

```bash
su -s /bin/sh -c "cp etc/zun/api-paste.ini /etc/zun" zun
```

### config files

1. Backup and sanitize **/etc/zun/zun.conf**

```bash
cp -p /etc/zun/zun.conf /etc/zun/zun.conf.bak
grep -Ev '^(#|$)' /etc/zun/zun.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/zun/zun.conf
```

2. Modify **/etc/zun/zun.conf**

```yaml
[DEFAULT]
#...
transport_url = rabbit://openstack:password123@controller:5672/

[api]
#...
host_ip = 10.10.10.11
port = 9517

[database]
#...
connection = mysql+pymysql://zun:password123@controller/zun

[keystone_auth]
memcached_servers = controller:11211
www_authenticate_uri = http://controller:5000
project_domain_name = default
project_name = service
user_domain_name = default
password = password123
username = zun
auth_url = http://controller:5000
auth_type = password
auth_version = v3
auth_protocol = http
service_token_roles_required = True
endpoint_type = internalURL

[keystone_authtoken]
#...
memcached_servers = controller:11211
www_authenticate_uri = http://controller:5000
project_domain_name = default
project_name = service
user_domain_name = default
password = password123
username = zun
auth_url = http://controller:5000
auth_type = password
auth_version = v3
auth_protocol = http
service_token_roles_required = True
endpoint_type = internalURL

[oslo_concurrency]
#...
lock_path = /var/lib/zun/tmp

[oslo_messaging_notifications]
#...
driver = messaging

[websocket_proxy]
#...
wsproxy_host = 10.10.10.11
wsproxy_port = 6784
base_url = ws://controller:6784/
```

3. Fix packaging bug

```
install -d /var/lib/zun/tmp -o zun -g zun
```

### Populate database

```bash
su -s /bin/sh -c "zun-db-manage upgrade" zun
```

### Create systemd service units

1. **/etc/systemd/system/zun-api.service**

```yaml
[Unit]
Description = OpenStack Container Service API

[Service]
ExecStart = /usr/local/bin/zun-api
User = zun

[Install]
WantedBy = multi-user.target
```

2. **/etc/systemd/system/zun-wsproxy.service**

```yaml
[Unit]
Description = OpenStack Container Service Websocket Proxy

[Service]
ExecStart = /usr/local/bin/zun-wsproxy
User = zun

[Install]
WantedBy = multi-user.target
```

### Start services

```bash
systemctl enable zun-api
systemctl enable zun-wsproxy

systemctl start zun-api
systemctl start zun-wsproxy
```

## 2. COMPUTE NODE

### Create zun user and directories

1. zun user

```
groupadd --system zun

useradd --home-dir "/var/lib/zun" \
--create-home \
--system \
--shell /bin/false \
-g zun \
zun
```

2. Config directory

```
mkdir -p /etc/zun
chown zun:zun /etc/zun
```

3. CNI Directory

```
mkdir -p /etc/cni/net.d
chown zun:zun /etc/cni/net.d
```

### Install packages

1. Install prereq packages

```
apt-get install git numactl -y
```

2. Clone and install zun

```
cd /var/lib/zun

git clone https://opendev.org/openstack/zun.git --branch stable/wallaby

chown -R zun:zun zun

cd zun

pip install -r requirements.txt

python3 setup.py install
```

3. Generate sample config file

```bash
 su -s /bin/sh -c "oslo-config-generator \
    --config-file etc/zun/zun-config-generator.conf" zun
su -s /bin/sh -c "cp etc/zun/zun.conf.sample \
    /etc/zun/zun.conf" zun
su -s /bin/sh -c "cp etc/zun/rootwrap.conf \
    /etc/zun/rootwrap.conf" zun
su -s /bin/sh -c "mkdir -p /etc/zun/rootwrap.d" zun
su -s /bin/sh -c "cp etc/zun/rootwrap.d/* \
    /etc/zun/rootwrap.d/" zun
su -s /bin/sh -c "cp etc/cni/net.d/* /etc/cni/net.d/" zun
```

4. Config sudoers for zun users

```bash
echo "zun ALL=(root) NOPASSWD: /usr/local/bin/zun-rootwrap \
    /etc/zun/rootwrap.conf *" | sudo tee /etc/sudoers.d/zun-rootwrap
```

### Config files

1. Backup and sanitize **/etc/zun/zun.conf**

```bash
cp -p /etc/zun/zun.conf /etc/zun/zun.conf.bak
grep -Ev '^(#|$)' /etc/zun/zun.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/zun/zun.conf
```

2. Edit **/etc/zun/zun.conf**

```yaml
[DEFAULT]
#...
transport_url = rabbit://openstack:RABBIT_PASS@controller
state_path = /var/lib/zun

[database]
#...
connection = mysql+pymysql://zun:ZUN_DBPASS@controller/zun

[keystone_auth]
memcached_servers = controller:11211
www_authenticate_uri = http://controller:5000
project_domain_name = default
project_name = service
user_domain_name = default
password = ZUN_PASS
username = zun
auth_url = http://controller:5000
auth_type = password
auth_version = v3
auth_protocol = http
service_token_roles_required = True
endpoint_type = internalURL

[keystone_authtoken]
#...
memcached_servers = controller:11211
www_authenticate_uri= http://controller:5000
project_domain_name = default
project_name = service
user_domain_name = default
password = ZUN_PASS
username = zun
auth_url = http://controller:5000
auth_type = password

[oslo_concurrency]
#...
lock_path = /var/lib/zun/tmp

[compute]
#...
host_shared_with_nova = true
```

### Config Docker and Kuryr

1. Create dir

```bash
mkdir -p /etc/systemd/system/docker.service.d
```

2. Create systemd unit file **/etc/systemd/system/docker.service.d/docker.conf**

```yaml
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --group zun -H tcp://compute1:2375 -H unix:///var/run/docker.sock --cluster-store etcd://controller:2379
```

3. Restart docker

```
systemctl daemon-reload
systemctl restart docker
```

4. Edit **/etc/kuryr/kuryr.conf**

```yaml
[DEFAULT]
#...
capability_scope = global
process_external_connectivity = False
```

5. Restart kuryr-libnetwork

```
systemctl restart kuryr-libnetwork
```

### Configure Containerd

1. Generate config

```
containerd config default > /etc/containerd/config.toml
```

2. Edit **/etc/containerd/config.toml**

```bash
getent group zun | cut -d: -f3
```

```yaml
[grpc]
#...
gid = ZUN_GROUP_ID
```

3. Restart containerd

```
systemctl restart containerd
```

### configure CNI

1. Download and install standard loopback plugin

```bash
mkdir -p /opt/cni/bin
curl -L https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz \
      | tar -C /opt/cni/bin -xzvf - ./loopback
```

2. Install Zun CNI plugin

```
install -o zun -m 0555 -D /usr/local/bin/zun-cni /opt/cni/bin/zun-cni
```

### Finalize install

1. Create systemd unit **/etc/systemd/system/zun-compute.service**

```yaml
[Unit]
Description = OpenStack Container Service Compute Agent

[Service]
ExecStart = /usr/local/bin/zun-compute
User = zun

[Install]
WantedBy = multi-user.target
```

2. Create systemd unit for zun cni **/etc/systemd/system/zun-cni-daemon.service**

```yaml
[Unit]
Description = OpenStack Container Service CNI daemon

[Service]
ExecStart = /usr/local/bin/zun-cni-daemon
User = zun

[Install]
WantedBy = multi-user.target
```

3. Enable and start zun-compute and zun-cni-daemon

```
systemctl enable zun-compute
systemctl start zun-compute

systemctl enable zun-cni-daemon
systemctl start zun-cni-daemon
```

### Verify Ops

1. Install client

```
apt install python3-zunclient -y
```

2. Source admin credentials

```
source ~/.adminrc
```

3. List service components

```
openstack appcontainer service list
```

4. Source demo credentials

```
source ~/.demorc
```

5. Run cirros container

```
NET_ID=$(openstack network list | awk '/ selfservice / { print $2 }')
openstack appcontainer run --name container --net network=$NETID cirros ping 8.8.8.8

openstack appcontainer list

openstack appcontainer exec --interactive container /bin/sh
ping -c 4 google.com
exit

openstack appcontainer stop container
openstack appcontainer delete container

openstack appcontainer run --name webcontainer --net network=$NETID --expose 80 httpd

openstack floating ip create provider

openstack appcontainer add floating ip <ip address>

# test web server container from browser

openstack appcontainer delete webcontainer

```
 