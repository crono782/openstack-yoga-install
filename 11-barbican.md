# Barbican install

> ![Barbican logo](/images/barbican.png)

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **barbican** database:

```sql
CREATE DATABASE barbican;
```

3. Grant proper access to **barbican** user:

```sql
GRANT ALL PRIVILEGES ON barbican.* TO 'barbican'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON barbican.* TO 'barbican'@'%' identified by 'password123';
exit
```

### Create Openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **barbican** user and add role:

```bash
openstack user create --domain default --password password123 barbican

openstack role add --project service --user barbican admin
```

* Create **creator** role and add to user:

```
openstack role create creator

openstack role add --project service --user barbican creator
```

* Create **barbican** service:

```bash
openstack service create --name barbican \
  --description "Key  Manager" key-manager
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  key-manager $i http://controller:9311; \
  done
```

### Install and configure componenets

1. Install packages:

```bash
apt install barbican-api barbican-keystone-listener barbican-worker python3-barbicanclient -y
```

2. Backup an sanitize **/etc/barbican/barbican.conf**:

```bash
cp -p /etc/barbican/barbican.conf /etc/barbican/barbican.conf.bak
grep -Ev '^(#|$)' /etc/barbican/barbican.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/barbican/barbican.conf
```

3. Create kek value

```
date|sha256sum|head -c 32|base64
```

3. Edit **/etc/barbican/barbican.conf** sections:

```yaml
[DEFAULT]
# ...
sql_connection = mysql+pymysql://barbican:password123@controller/barbican
transport_url = rabbit://openstack:password123@controller

[keystone_authtoken]
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = barbican
password = password123

[secretstore]
#...
enabled_secretstore_plugins = store_crypto

[crypto]
#...
enabled_crypto_plugins = simple_crypto

[simple_crypto_plugin]
# the kek should be a 32-byte value which is base64 encoded
kek = 'YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY='
```

### populate database

```
su -s /bin/sh -c "barbican-manage db upgrade" barbican
```

### finalize install

```
systemctl enable barbican-keystone-listener
systemctl enable barbican-worker
service barbican-keystone-listener restart
service barbican-worker restart
service apache2 restart
```

### verify ops

```
openstack secret store --name mysecret --payload j4=]d21

export SECRET_HREF=$(openstack secret list --name mysecret -c 'Secret href' -f value)

openstack secret get $SECRET_HREF

openstack secret get $SECRET_HREF --payload

openstack secret delete $SECRET_HREF

unset SECRET_HREF
```







