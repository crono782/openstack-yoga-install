# Horizon

> ![Horizon logo](/images/horizon.png)

## Install packages

```bash
apt install openstack-dashboard -y
```

## Modify config file **/etc/openstack-dashboard/local_settings.py**:

```yaml
OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*']

# ALLOWED_HOSTS - ['host1', 'host1']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST

# IF ONLY PROVIDER NEWORKS
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "TIME_ZONE"
```

* Modify file **/etc/apache2/conf-available/openstack-dashboard.conf** if line is not present:

```yaml
WSGIApplicationGroup %{GLOBAL}
```

* Finalize install:

```bash
systemctl reload apache2
```


