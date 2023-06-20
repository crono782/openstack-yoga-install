# Placement Install

> -

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **placement** database:

```sql
CREATE DATABASE placement;
```

3. Grant proper access to **placement** user and exit:

```sql
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' identified by 'password123';
exit
```

### Create openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **placement** user and add role:

```bash
openstack user create --domain default --password password123 placement

openstack role add --project service --user placement admin
```

* Create **placement** service:

```bash
openstack service create --name placement \
  --description "Placement API" placement
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  placement $i http://controller:8778; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install placement-api python3-osc-placement -y
```

2. Backup an sanitize **/etc/placement/placement.conf**:

```bash
cp -p /etc/placement/placement.conf /etc/placement/placement.conf.bak
grep -Ev '^(#|$)' /etc/placement/placement.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/placement/placement.conf
```

2. Edit **/etc/placement/placement.conf** sections:

```yaml
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
password = password123

[placement_database]
# ...
# remove other connections
connection = mysql+pymysql://placement:password123@controller/placement
```

3. Populate database:

```bash
su -s /bin/sh -c "placement-manage db sync" placement
```

4. Restart web server:

```bash
service apache2 restart
```

### Verify ops

1. Source .adminrc

```bash
source .adminrc
```

2. Perform status checks:

```bash
placement-status upgrade check

openstack --os-placement-api-version 1.2 resource class list --sort-column name

openstack --os-placement-api-version 1.6 trait list --sort-column name
```