# Neutron Install

> ![Neutron logo](/images/neutron.png)

## 1: CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **neutron** database:

```sql
CREATE DATABASE neutron;
```

3. Grant proper access to **neutron** user:

```sql
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' identified by 'password123';
exit
```

### Create Openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **neutron** user and add role:

```bash
openstack user create --domain default --password password123 neutron

openstack role add --project service --user neutron admin
```

* Create **neutron** service:

```bash
openstack service create --name neutron \
  --description "Openstack Networking" network
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  network $i http://controller:9696; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install neutron-server neutron-plugin-ml2 -y
```

2. Backup an sanitize **/etc/neutron/neutron.conf**:

```bash
cp -p /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
grep -Ev '^(#|$)' /etc/neutron/neutron.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/neutron.conf
```

3. Edit **/etc/neutron/neutron.conf** sections:

```yaml
[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
# ...
# remove other connections
connection = mysql+pymysql://neutron:password123@controller/neutron

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password123

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = password123

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

4. Remove unused sqlite database file:

```bash
rm -f /var/lib/neutron/neutron.sqlite
```

5. Fix packaging bug:

```bash
install -d /var/lib/neutron/tmp -o neutron -g neutron
```

#### Configure ML2 plugin

1. Backup an sanitize **/etc/neutron/plugins/ml2/ml2_conf.ini**:

```bash
cp -p /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bak
grep -Ev '^(#|$)' /etc/neutron/plugins/ml2/ml2_conf.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/plugins/ml2/ml2_conf.ini
```

2. Edit **/etc/neutron/plugins/ml2/ml2_conf.ini** sections:

```yaml
[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true
```

#### Configure nova to use neutron

1. Edit **/etc/nova/nova.conf** sections:

```yaml
[neutron]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = password123
service_metadata_proxy = true
metadata_proxy_shared_secret = s3cr3t_m3tadat4
```

### Finalize installation

1. Populate database:

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

2. Restart compute API service:

```bash
service nova-api restart
```

3. Restart Networking API service

```bash
service neutron-server restart
```

## 2: NETWORK NODE

### Install and configure componenets

1. Install packages:

```bash
apt install neutron-server \
neutron-linuxbridge-agent neutron-l3-agent \
neutron-dhcp-agent neutron-metadata-agent \
python3-neutron-fwaas -y
```

2. Backup an sanitize **/etc/neutron/neutron.conf**:

```bash
cp -p /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
grep -Ev '^(#|$)' /etc/neutron/neutron.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/neutron.conf
```

3. Edit **/etc/neutron/neutron.conf** sections:

```yaml
[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password123

[oslo_concurrency]
# ...
# make sure this dir exists
lock_path = /var/lib/neutron/tmp
```

#### Configure Linux bridge agent

1. Backup an sanitize **/etc/neutron/plugins/ml2/linuxbridge_agent.ini**:

```bash
cp -p /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
grep -Ev '^(#|$)' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

2. Edit **/etc/neutron/plugins/ml2/linuxbridge_agent.ini** sections:

```yaml
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true
```

3. Ensure network bridge filters enabled (br_netfilter kmod):

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

#### Configure L3 agent

1. Backup an sanitize **/etc/neutron/l3_agent.ini**:

```bash
cp -p  /etc/neutron/l3_agent.ini  /etc/neutron/l3_agent.ini.bak
grep -Ev '^(#|$)'  /etc/neutron/l3_agent.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 >  /etc/neutron/l3_agent.ini
```

2. Edit **/etc/neutron/l3_agent.ini** sections:

```yaml
[DEFAULT]
# ...
interface_driver = linuxbridge
```

#### Configure DHCP agent

1. Backup an sanitize **/etc/neutron/dhcp_agent.ini**:

```bash
cp -p  /etc/neutron/dhcp_agent.ini  /etc/neutron/dhcp_agent.ini.bak
grep -Ev '^(#|$)'  /etc/neutron/dhcp_agent.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 >  /etc/neutron/dhcp_agent.ini
```

2. Edit **/etc/neutron/dhcp_agent.ini** sections:

```yaml
[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

#### Configure Metadata agent

1. Backup an sanitize ** /etc/neutron/metadata_agent.ini**:

```bash
cp -p   /etc/neutron/metadata_agent.ini   /etc/neutron/metadata_agent.ini.bak
grep -Ev '^(#|$)'   /etc/neutron/metadata_agent.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 >   /etc/neutron/metadata_agent.ini
```

2. Edit ** /etc/neutron/metadata_agent.ini** sections:

```yaml
[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = s3cr3t_m3tadat4
```

3. Fix packaging bug:

```bash
install -d /var/lib/neutron/tmp -o neutron -g neutron
```

### Start services:

```bash
service neutron-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
service neutron-l3-agent restart
```

## 3: COMPUTE NODE

### Install and configure components

#### Configure common component

1. Install packages:

```bash
apt install neutron-linuxbridge-agent -y
```

2. Backup an sanitize **/etc/neutron/neutron.conf**:

```bash
cp -p /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
grep -Ev '^(#|$)' /etc/neutron/neutron.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/neutron.conf
```

3. Edit **/etc/neutron/neutron.conf** sections:

```yaml
[DEFAULT]
# ...
transport_url = rabbit://openstack:password123@controller
auth_strategy = keystone

[database]
# NOTHING

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = password123

[oslo_concurrency]
# ...
lock_path = /var/lib/neutron/tmp
```

4. Fix packaging bug:

```bash
install -d /var/lib/neutron/tmp -o neutron -g neutron
```

#### Configure Linux bridge agent

1. Backup an sanitize **/etc/neutron/plugins/ml2/linuxbridge_agent.ini**:

```bash
cp -p /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
grep -Ev '^(#|$)' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

2. Edit **/etc/neutron/plugins/ml2/linuxbridge_agent.ini** sections:

```yaml
[linux_bridge]
physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

[vxlan]
enable_vxlan = true
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

* TODO: Is phys_interface_mappings required here?

3. Ensure network bridge filters enabled (br_netfilter kmod):

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

### Restart services

* Restart compute service:

```bash
service nova-compute restart
```

## 4: CONTROLLER NODE

### Verify ops

1. Source .adminrc

```bash
source .adminrc
```

2. Show network agents (should show 4 on network and 1 on compute):

```bash
openstack network agent list
```








