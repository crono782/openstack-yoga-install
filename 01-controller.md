# Controller setup

# 1. CONTROLLER NODE

### MySQL

1. Install mysql packages:

```bash
apt install mariadb-server python3-pymysql -y
```

2. Create and edit **/etc/mysql/mariadb.conf.d/99-openstack.cnf**

```yaml
[mysqld]
bind-address = 10.10.10.11
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

3. Finalize MYSQL installation

```bash
systemctl enable mariadb
systemctl restart mariadb
```

4. Run mysql setup:

```bash
mysql_secure_installation
```

### RabbitMQ

1. Install rabbitmq packages:

```bash
apt install rabbitmq-server -y
```

2. Add **openstack** user to rabbitmq:

```bash
rabbitmqctl add_user openstack password123
```

3. Apply permissions for **openstack** rabbitmq user:

```bash
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### Memcached

1. Install memcached packages:

```bash
apt install memcached python3-memcache -y
```

2. Edit **/etc/memcached.conf** and change following setting:

```yaml
-l 10.10.10.11
```

3. Finalize installation

```bash
systemctl enable memcached
systemctl restart memcached
```

## Etcd

1. Install etcd packages:

```bash
apt install etcd -y
```

2. Edit **/etc/default/etcd** and change following setting:

```yaml
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.10.10.11:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.10.10.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.11:2379"
ETCD_LISTEN_PEER_URLS="http://10.10.10.11:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.10.10.11:2379"
```

1. Enable and restart etcd service:

```bash
systemctl enable etcd
systemctl restart etcd
```