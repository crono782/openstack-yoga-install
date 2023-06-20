# KVM Guest setup

## QCOW2 Disk Creation

* Example showing how to create a thin qcow2 disk

```bash
qemu-img create -f qcow2 VOLUME_NAME.qcow2 VOLUME_SIZE_GB
```

* Example showing how to create a qcow2 disk with backing volume

```bash
qemu-img create -f qcow2 -F qcow2 -b BACKING_VOLUME.qcow2 NEW_VOLUME_NAME.qcow2
```

## VM Installs

* Example controller vm. no special setup

```bash
virt-install -n os-controller --os-type Linux --os-variant ubuntu20.04 --ram 8192 --vcpus 2 --import --disk /data/images/os-controller.qcow2 --network bridge:br-mgt --graphics vnc --noautoconsole
```

* Example compute vm. host cpu passthrough for nested kvm and tenant network

```bash
virt-install -n os-compute --os-type Linux --os-variant ubuntu20.04 --ram 16384 --vcpus 2 --cpu host-passthrough --import --disk /data/images/os-compute.qcow2 --network bridge:br-mgt --network bridge:br-tnt --graphics vnc --noautoconsole
```

* Example network vm. tenant and provider networks

```bash
virt-install -n os-network --os-type Linux --os-variant ubuntu20.04 --ram 2048 --vcpus 1 --import --disk /data/images/os-network.qcow2 --network bridge:br-mgt --network bridge:br-tnt --network bridge:br-ext --graphics vnc --noautoconsole
```

* Example ceph, block, or object vms. multiple disks

```bash
virt-install --name os-ceph --os-type Linux --os-variant centos7.0 --ram 4096 --vcpus 1 --import --disk /data/images/os-ceph.qcow2 --disk /data/disks/os-ceph-data1.qcow2 --disk /data/disks/os-ceph-data2.qcow2 --disk /data/disks/os-ceph-data3.qcow2 --disk /data/disks/os-ceph-data4.qcow2 --network bridge:br0 --graphics vnc --noautoconsole
```

## Disable cloud-init

```bash
touch /etc/cloud/cloud-init.disabled
```

## Add console output

1.  Add the following to **/etc/default/grub**:

```bash
GRUB_CMDLINE_LINUX="console=ttyS0"
```

2. Rebuild grub:

```bash
update-grub
```

## Guest bootstrapping

1. Set host name

```bash
hostnamectl set-hostname HOSTNAME
```

2. Set hosts file

```bash
sed -i 's/OLDHOSTNAME/NEWHOSTNAME/g' /etc/hosts
```

3. Set up network (refer to netplan samples)

```bash
vi /etc/netplan/00-installer-config.yaml
netplan apply
```

4. Set up timezone

```bash
timedatectl set-timezone TIMEZONE
```

5. Install NTP

```bash
apt install chrony -y
```

6. Disable AppArmor

```bash
systemctl disable apparmor
```

7. Disable ufw (firewall)

```bash
ufw disable
```

8. Reboot guest

```bash
reboot
```

## Netplan examples

* Simple netplan example for one interface on management network

```yaml
network:
  ethernets:
    enp1s0:
      addresses:
      - 10.10.10.11/24
      gateway4: 10.10.10.1
      nameservers:
        addresses:
        - 8.8.8.8
        search: []
  version: 2
```

* More complex example showing two networks. Note that some trickery needed to avoid duplicate default routes

```yaml
network:
  ethernets:
    enp1s0:
      addresses:
        - 10.10.10.11/24
      gateway4: 10.10.10.1
      nameservers:
        addresses:
        - 8.8.8.8
        search: []
    enp2s0:
      addresses:
        - 10.10.20.11/24
      routes:
        - to: 10.10.20.0/24
          via: 10.10.20.1
          metric: 500
          table: 1
      routing-policy:
        - from: 10.10.20.11
          table: 1
  version: 2
```

* Even more complex example showing three networks, one of which has no address. Note that some trickery needed to avoid duplicate default routes and a quick trick to have an interface with no config.

```yaml
network:
  ethernets:
    enp1s0:
      addresses:
        - 10.10.10.11/24
      gateway4: 10.10.10.1
      nameservers:
        addresses:
        - 8.8.8.8
        search: []
    enp2s0:
      addresses:
        - 10.10.20.11/24
      routes:
        - to: 10.10.20.0/24
          via: 10.10.20.1
          metric: 500
          table: 1
      routing-policy:
        - from: 10.10.20.11
          table: 1
    enp3s0: {}
  version: 2
```