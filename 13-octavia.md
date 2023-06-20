# Octavia Install

> ![Octavia logo](/images/octavia.png)

* TODO: Split into multinode install

## 1. CONTROLLER NODE

### Database setup

```bash
mysql
```

```bash
CREATE DATABASE octavia;
```

```bash
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'localhost' IDENTIFIED BY 'password123';
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'%' IDENTIFIED BY 'password123';
exit
```

### Create Openstack resources

```bash
source .adminrc
```

```bash
openstack user create --domain default --password password123 octavia
```

```bash
openstack role add --project service --user octavia admin
```

```bash
openstack service create --name octavia --description "OpenStack Octavia" load-balancer
```

```bash
for i in public internal admin; do \
  openstack endpoint create --region RegionOne \
  load-balancer $i http://controller:9876; done
```

### Create octavia rc file

```bash
cat << EOF >> ~/.octaviarc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=service
export OS_USERNAME=octavia
export OS_PASSWORD=password123
export OS_AUTH_URL=http://controller:5000
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export OS_VOLUME_API_VERSION=3
EOF
```

### Install packages

```bash
apt install octavia-api octavia-health-manager octavia-housekeeping octavia-worker python3-octavia python3-octaviaclient -y
```

### Build amphora image

* Uses build script. More control and produces smaller image. Alternative method using snap in appendix.

```bash

git clone https://opendev.org/openstack/octavia.git --branch stable/wallaby

# Use your preferred/system version of python
apt install python3.X-venv qemu-utils git kpartx debootstrap -y 

python3 -m venv octavia_disk_image_create

source octavia_disk_image_create/bin/activate

cd octavia/diskimage-create

pip install -r requirements.txt

./diskimage-create.sh
```

### Upload amphora image

```bash
source ~/.octavia-openrc
```

```bash
openstack image create --disk-format qcow2 --container-format bare \
  --private --tag amphora --file amphora--x64-haproxy.qcow2 amphora-x64-haproxy
```

### Create amphora flavor (may need 5G size if used snap builder)

```bash
openstack flavor create --id 200 --vcpus 1 --ram 1024 \
  --disk 2 "lb.m1.small" --private
```

### Create certs

```bash
sudo mkdir -p /etc/octavia/certs/private
sudo chmod 755 /etc/octavia -R
cd ~
# only need to clone if didn't do it before
git clone https://opendev.org/openstack/octavia.git --branch stable/wallaby
cd octavia/bin/
source create_dual_intermediate_CA.sh
sudo cp -p etc/octavia/certs/server_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca-chain.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/server_ca.key.pem /etc/octavia/certs/private
sudo cp -p etc/octavia/certs/client_ca.cert.pem /etc/octavia/certs
sudo cp -p etc/octavia/certs/client.cert-and-key.pem /etc/octavia/certs/private
chown -R octavia:octavia /etc/octavia/certs
```

### Create sec group and rules

```bash
source ~/.octavia-openrc
```

```bash
openstack security group create lb-mgmt-sec-grp
openstack security group rule create --protocol icmp lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 22 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 80 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 443 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 9443 lb-mgmt-sec-grp
openstack security group create lb-health-mgr-sec-grp
openstack security group rule create --protocol udp --dst-port 5555 lb-health-mgr-sec-grp
```

### Create keypair for testing (optional)

```
openstack keypair create octavia-mgmt
```

### Network setup

1. Create LB network

* Different method than show in docs or devstack. This doesn't use dhclient. Works better for me and doesn't lock my system.

```
NETID=$(openstack network show lb-mgmt-net -c id -f value)

BRNAME=brq$(echo $NETID|cut -c 1-11)

ip link add o-hm0 type veth peer name o-bhm0

ip link set dev o-hm0 address $MGMT_PORT_MAC

ip link set o-hm0 up
ip link set o-bhm0 up

ip addr add $OCTAVIA_MGMT_PORT_IP/24 dev o-hm0

brctl addif $BRNAME o-bhm0

iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
```

2. Create network startup files.

* Create **/opt/octavia-interface.sh** (sub values for real ones)

```bash
#!/bin/bash

set -ex

MAC=$MGMT_PORT_MAC
BRNAME=$BRNAME

if [ "$1" == "start" ]; then
  ip link add o-hm0 type veth peer name o-bhm0
  ip link set dev o-hm0 address $MAC
  ip link set o-hm0 up
  ip link set o-bhm0 up
  ip addr add $OCTAVIA_MGMT_PORT/24 dev o-hm0
  brctl addif $BRNAME o-bhm0
  iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
elif [ "$1" == "stop" ]; then
  ip link del o-hm0
else
  brctl show $BRNAME
  ip a s dev o-hm0
fi
```

* Set permissions

```bash
chmod o+x /opt/octavia-interface.sh
```

* Create systemd unit file **/etc/systemd/system/octavia-interface.service**

```yaml
[Unit]
Description=Octavia Interface Creator

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/opt/octavia-interface.sh start
ExecStop=/opt/octavia-interface.sh stop
```

* Interface service target cannot start until Neutron has had sufficient time to process and bring up networks. May need to adjust timer to suit. Configure a systemd timer unit file **/etc/systemd/system/octavia-interface.timer**

```yaml
[Unit]
Description=Time for Octavia Interface Creator
Requires=neutron-linuxbridge-agent.service
After=neutron-linuxbridge-agent.service

[Timer]
OnBootSec=5min

[Install]
WantedBy=timers.target
```

* Disable service unit and enable timer unit

```bash
systemctl daemon-reload
systemctl disable octavia-interface.service
systemctl enable  octavia-interface.timer
```

### config files

1. Backup and sanitize **/etc/octavia/octavia.conf**

```bash
cp -p /etc/octavia/octavia.conf /etc/octavia/octavia.conf.bak
grep -Ev '^(#|$)' /etc/octavia/octavia.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/octavia/octavia.conf
```

2. Modify **/etc/octavia/octavia.conf**

```yaml
[DEFAULT]
transport_url = rabbit://openstack:password123@controller:5672

[api_settings]
bind_host = 0.0.0.0
bind_port = 9876
auth_strategy = keystone
api_base_uri = http://controller:9876

[certificates]
server_certs_key_passphrase = insecure-key-do-not-use-this-key
ca_private_key_passphrase = not-secure-passphrase
ca_private_key = /etc/octavia/certs/private/server_ca.key.pem
ca_certificate = /etc/octavia/certs/server_ca.cert.pem

[controller_worker]
client_ca = /etc/octavia/certs/client_ca.cert.pem

amp_image_owner_id = <service project id>
amp_image_tag = amphora
# key only for testing
amp_ssh_key_name = octavia-mgmt
amp_secgroup_list = <lb-mgmt-sec-grp-id>
amp_boot_network_list = <lb-mgmt-net-id>
amp_flavor_id = 200
network_driver = allowed_address_pairs_driver
compute_driver = compute_nova_driver
amphora_driver = amphora_haproxy_rest_driver

[database]
connection = mysql+pymysql://octavia:password123@controller/octavia

[haproxy_amphora]
server_ca = /etc/octavia/certs/server_ca-chain.cert.pem
client_cert = /etc/octavia/certs/private/client.cert-and-key.pem

[health_manager]
# this option missing from docs
heartbeat_key = insecure-key
bind_ip = 0.0.0.0
bind_port = 5555
controller_ip_port_list = 172.16.0.2:5555

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = password123

[oslo_messaging]
topic = octavia_prov

[service_auth]
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = password123
```

### Run database migrations

```
octavia-db-manage --config-file /etc/octavia/octavia.conf upgrade head
```

### Finalize install

```
systemctl restart octavia-api octavia-health-manager octavia-housekeeping octavia-worker
```

### Install horizon dashboard for octavia

```
apt intall python3-octavia-dashboard -y

service apache2 restart
```

### Create loadbalancer

1. Create two instances responding to http port 80 traffic. Be sure to allow port 80 on local private subnet

```bash
# can use this snippet on cirros instances since no web server available
while true; do echo -e "HTTP/1.1 200 OK\r\n\r\n$(hostname)" | sudo nc -l -p 80; done
```

2. Create loadbalancer

```
openstack loadbalancer create --name lb01 --vip-subnet-id private-subnet
```

2. Create listener

```
openstack loadbalancer listener create --name listener01 --protocol TCP --protocol-port 80 lb01
```

3. Create pool

```
openstack loadbalancer pool create --name pool01 --lb-algorithm ROUND_ROBIN --listener listener01 --protocol TCP
```

4. Create pool members

```
openstack loadbalancer member create --subnet-id private-subnet --address <server ip> --protocol-port 80 pool01

openstack loadbalancer member create --subnet-id private-subnet --address <server ip> --protocol-port 80 pool01
```

5. Create floating ip for load balancer

```
openstack floating ip create provider
```

6. Assign floating ip to loadbalancer vip port

```
VIPPORT=$(openstack loadbalancer show lb01 | grep vip_port_id | awk {'print $4'})
openstack floating ip set --port $VIPPORT <floating ip>
```

7. Test from public side

Use web browser or curl to hit floating ip/vip address. should show switching from one lb member to another


### Appendix

* Alternative method of building amphora using a snap. Easier, but produces larger image

```bash
snap install octavia-diskimage-retrofit --beta --devmode

cd /var/snap/octavia-diskimage-retrofit/common/tmp

wget https://cloud-images.ubuntu.com/minimal/releases/focal/release/ubuntu-20.04-minimal-cloudimg-amd64.img

octavia-diskimage-retrofit ubuntu-20.04-minimal-cloudimg-amd64.img ubuntu-amphora-haproxy-amd64.qcow2
```

* This is the method in documentation AND devstack for building the network injector. Locks my system though. Only including here for doc purposes. Don't use this!

* Create dhcp config

```
sudo mkdir -m755 -p /etc/dhcp/octavia
sudo cp octavia/etc/dhcp/dhclient.conf /etc/dhcp/octavia
```

* Build network injector

```
OCTAVIA_MGMT_SUBNET=172.16.0.0/24
OCTAVIA_MGMT_SUBNET_START=172.16.0.100
OCTAVIA_MGMT_SUBNET_END=172.16.0.254
OCTAVIA_MGMT_PORT_IP=172.16.0.2

openstack network create lb-mgmt-net
openstack subnet create --subnet-range $OCTAVIA_MGMT_SUBNET --allocation-pool \
  start=$OCTAVIA_MGMT_SUBNET_START,end=$OCTAVIA_MGMT_SUBNET_END \
  --network lb-mgmt-net lb-mgmt-subnet

SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id)
PORT_FIXED_IP="--fixed-ip subnet=$SUBNET_ID,ip-address=$OCTAVIA_MGMT_PORT_IP"

MGMT_PORT_ID=$(openstack port create --security-group \
  lb-health-mgr-sec-grp --device-owner Octavia:health-mgr \
  --host=$(hostname) -c id -f value --network lb-mgmt-net \
  $PORT_FIXED_IP octavia-health-manager-listen-port)

MGMT_PORT_MAC=$(openstack port show -c mac_address -f value \
  $MGMT_PORT_ID)

sudo ip link add o-hm0 type veth peer name o-bhm0
NETID=$(openstack network show lb-mgmt-net -c id -f value)
BRNAME=brq$(echo $NETID|cut -c 1-11)
sudo brctl addif $BRNAME o-bhm0
sudo ip link set o-bhm0 up

sudo ip link set dev o-hm0 address $MGMT_PORT_MAC
sudo iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
sudo dhclient -v o-hm0 -cf /etc/dhcp/octavia
```