# Heat install

> ![Heat logo](/images/heat.png)

## 1. CONTROLLER NODE

### Database setup

1. Access database as root:

```bash
mysql
```

2. Create **heat** database:

```sql
CREATE DATABASE heat;
```

3. Grant proper access to **heat** user and exit:

```sql
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' identified by 'password123';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' identified by 'password123';
exit
```

### Create openstack objects

1. Source .adminrc

```bash
source .adminrc
```

2. Create service and creds

* Create **heat** user and add role:

```bash
openstack user create --domain default --password password123 heat

openstack role add --project service --user heat admin
```

* Create **heat** services:

```bash
openstack service create --name heat \
  --description "Orchestration" orchestration

openstack service create --name heat-cfn \
  --description "Orchestration" cloudformation
```

3. Create service API endpoints:

```bash
for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  orchestration $i http://controller:8004/v1/%\(tenant_id\)s; \
  done

for i in public internal admin; \
  do openstack endpoint create --region RegionOne \
  cloudformation $i http://controller:8000/v1; \
  done
```

### Create heat domain

```
openstack domain create --description "Stack projects and users" heat

openstack user create --domain heat --password password123 heat_domain_admin

openstack role add --domain heat --user-domain heat --user heat_domain_admin admin

openstack role create heat_stack_owner

openstack role add --project demoproject --user demouser heat_stack_owner

openstack role create heat_stack_user
```

### Install and configure components

1. Install packages:

```bash
apt-get install heat-api heat-api-cfn heat-engine \
python3-vitrageclient python3-zunclient -y
```


2. Backup an sanitize **/etc/heat/heat.conf**:

```bash
cp -p /etc/heat/heat.conf /etc/heat/heat.conf.bak
grep -Ev '^(#|$)' /etc/heat/heat.conf.bak|sed '/^\[.*]/i \ '|tail -n +2 > /etc/heat/heat.conf
```

2. Edit **/etc/heat/heat.conf** sections:

```yaml
[DEFAULT]
#...
transport_url = rabbit://openstack:password123@controller
heat_metadata_server_url = http://controller:8000
heat_waitcondition_server_url = http://controller:8000/v1/waitcondition
stack_domain_admin = heat_domain_admin
stack_domain_admin_password = password123
stack_user_domain_name = heat

[database]
#...
connection = mysql+pymysql://heat:password123@controller/heat

[keystone_authtoken]
#...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = password123

[trustee]
#...
auth_type = password
auth_url = http://controller:5000
username = heat
password = password123
user_domain_name = default

[clients_keystone]
#...
auth_uri = http://controller:5000
```

### Finalize install

1. Populate database

```
su -s /bin/sh -c "heat-manage db_sync" heat
```

2. Restart services

```
service heat-api restart
service heat-api-cfn restart
service heat-engine restart
```

### Verify ops

* Verify service list

```
openstack orchestration service list
```

### Install horizon UI

1. install packages

```
apt install python3-heat-dashboard -y
```

2. Copy plugin setup files

```bash
cp /usr/lib/python3/dist-packages/heat_dashboard/enabled/_[0-9]*.py /usr/share/openstack-dashboard/openstack_dashboard/local/enabled/
```

3. Add policy to horizon settings **/etc/openstack-dashboard/local_settings.py**

```yaml
POLICY_FILES = {
    'orchestration': 'heat_policy.json'
}
```

4. restart apache

```
service apache2 restart
```

### Create basic template to **demo-template.yml**

```yaml
heat_template_version: 2021-04-16

description: >
    Simple template to deploy a single compute instance.
    Nothing special, just for testing.

parameters:
  NetID:
    type: string
    description: Network ID to use for the instance.

resources:
  my_key:
    type: OS::Nova::KeyPair
    properties:
      name: my_key
      save_private_key: true

  my_instance:
    type: OS::Nova::Server
    properties:
      key_name: { get_resource: my_key }
      image: cirros
      flavor: m1.micro
      networks:
        - network: { get_param: NetID }

outputs:
  private_key:
    description: Private key
    value: { get_attr: [ my_key, private_key ] }
  instance_name:
    description: Name of the instance
    value: { get_attr: [ my_instance, name ]}
  instance_ip:
    description: IP address of the instance.
    value: { get_attr: [ my_instance, first_address ] }
```

3. Create a stack

```
openstack stack create -t demo-template.yml --parameter "NetID=selfservice" stack
```

4. Check output

```
openstack stack list

openstack stack output show --all stack

openstack server list

openstack stack delete --yes stack
```