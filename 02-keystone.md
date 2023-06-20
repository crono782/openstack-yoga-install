# Keystone Install

> ![Keystone logo](/images/keystone.png)

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **keystone** database:

```sql
CREATE DATABASE keystone;
```

3. Grant proper access to **keystone** user and exit:

```sql
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' identified by 'password123';
exit
```

### Install and configure

1. Install packages:

```bash
apt install keystone -y
```

2. Backup an sanitize **/etc/keystone/keystone.conf**:

```bash
cp -p /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
grep -Ev '^(#|$)' /etc/keystone/keystone.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/keystone/keystone.conf
```

3. Edit **/etc/keystone/keystone.conf** sections:

```yaml
[database]
# ...
connection = mysql+pymysql://keystone:password123@controller/keystone

[token]
# ...
provider = fernet
```

4. Populate database:

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

### Bootstrap Keystone

1. Initialize fernet:

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

2. Bootstrap identity service:

```bash
keystone-manage bootstrap --bootstrap-password password123 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

### Configure Apache HTTP server

1. Edit **/etc/apache2/apache2.conf** and add **ServerName** option:

```yaml
ServerName controller
```

2. Restart apache service:

```bash
service apache2 restart
```

### Create admin RC file

* Create **~/.adminrc** file:

```yaml
export OS_USERNAME=admin
export OS_PASSWORD=password123
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

### Create domain, projects, users, and roles

1. Source .adminrc

```bash
source ~/.adminrc
```

2. Create service project

```bash
openstack project create --domain default \
  --description "Service Project" service
```

3. Create demo project, user, and role

```bash
openstack project create --domain default \
  --description "Demo Project" demoproject

openstack user create --domain default \
  --password password123 demouser
```

4. Assign member role demouser in demo project

```bash
openstack role add --project demoproject --user demouser member
```

### Create demo RC file

* Create **~/.demorc** file:

```yaml
export OS_USERNAME=demouser
export OS_PASSWORD=password123
export OS_PROJECT_NAME=demoproject
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```