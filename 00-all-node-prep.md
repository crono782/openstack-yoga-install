# All Node Prep

## Openstack Common Packages

1. Add archive for Wallaby

```bash
add-apt-repository cloud-archive:wallaby -y
```

2. Install Openstack client

```bash
apt install python3-openstackclient -y
```

3. Add nodes to **/etc/hosts** file:

```yaml
# openstack nodes
10.10.10.11 controller
10.10.10.12 compute
10.10.10.13 network
```
