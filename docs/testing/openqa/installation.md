# Installation

## Creating multi machine worker cluster

Firewall settings on workers:

```bash
firewall-cmd --add-port={5991-5994,20000-20100}/tcp --permanent
firewall-cmd --reload
```

Firewall settings on master:

```bash
firewall-cmd --add-port=5991-5994/tcp --permanent
firewall-cmd --reload
```

SELinux on master:

```bash
setsebool -P domain_can_mmap_files 1
```

NFS shares in fstab:

```bash
<NFShost>:/openqa_shared/hdd /var/lib/openqa/share/factory/hdd nfs nfsvers=3 0 0
<NFShost>:/openqa_shared/iso /var/lib/openqa/share/factory/iso nfs nfsvers=3 0 0
<NFShost>:/openqa_shared/tests /var/lib/openqa/share/tests nfs nfsvers=3 0 0
```

## Connect VNC client to QEMU workers

For Windows I recommend the VNC Viewer by RealVNC, for Linux Remmina

Windows Config:

- Disable `Authenticate using single sign-on if possible`
- Disable `Authenticate usin a smartcard or certificate store if possible`
- Set picture quality to `High`

Especially without the the picture quality setting it is sometimes unable to connect.

Linux Config:

- Protocol: Remmina VNC Plugin

Connection will only be possible when a test is running.

## How to run a single test suite inside of a product

```bash
sudo openqa-cli api -X POST isos ISO=<iso-image> ARCH=x86_64 DISTRI=rocky FLAVOR=<product> VERSION=<version> BUILD=-<product>-$(date +%Y%m%d.%H%M%S).0 TEST=<test suite>
```

## Setup multi-machine multi-vm networking for fedora

[https://github.com/os-autoinst/openQA/blob/master/docs/Networking.asciidoc](https://github.com/os-autoinst/openQA/blob/master/docs/Networking.asciidoc)

[https://fedoraproject.org/wiki/OpenQA_advanced_network_guide](https://fedoraproject.org/wiki/OpenQA_advanced_network_guide)

### Master configuration

- `dnf install os-autoinst-openvswitch tunctl network-scripts`
- `vi /etc/sysconfig/os-autoinst-openvswitch`

with the content:

```ini
OS_AUTOINST_BRIDGE_LOCAL_IP=172.16.2.2
OS_AUTOINST_BRIDGE_REWRITE_TARGET=172.17.0.0
```

- `vi /etc/sysconfig/network-scripts/ifcfg-br0`

with the content:

```ini
DEVICETYPE='ovs'
TYPE='OVSBridge'
BOOTPROTO='static'
IPADDR='172.16.2.2'
NETMASK='255.254.0.0'
DEVICE=br0
STP=off
ONBOOT='yes'
NAME='br0'
HOTPLUG='no'
```

For every openqa-worker 1 tap interface (worker@1 = tap0, worker@2 = tap1, ...)

- `vi /etc/sysconfig/network-scripts/ifcfg-tap0`

with the content:

```ini
DEVICETYPE='ovs'
TYPE='OVSPort'
OVS_BRIDGE='br0'
DEVICE='tap0'
ONBOOT='yes'
BOOTPROTO='none'
HOTPLUG='no'
```

- `vi /sbin/ifup-pre-local`

In the last line, configure a GRE tunnel for every additional host which will be connected to the master

(add multiple lines of the gre tunnel creation if needed)

with the content:

```sh
#!/bin/sh

if=$(echo "$1" | sed -e 's,ifcfg-,,')
iftype=$(echo "$if" | sed -e 's,[0-9]\+$,,')

# if the interface being brought up is tap[n], create
# the tap device first
if [ "$iftype" == "tap" ]; then
    tunctl -u _openqa-worker -p -t "$if"
fi

# if the interface being brough up is br0, create
# the gre tunnels
if [ "$if" == "br0" ]; then
    ovs-vsctl set bridge br0 stp_enable=true
    ovs-vsctl --may-exist add-port br0 gre<counting-number> -- set interface gre<counting-number> type=gre options:remote_ip=<ip-of-external-worker-host>
fi
```

- `chmod ug+x /sbin/ifup-pre-local`
- `firewall-cmd --permanent --zone=internal --add-interface=br0`
- `firewall-cmd --permanent --zone=public --add-masquerade`
- `firewall-cmd --permanent --zone=internal --add-masquerade`
- `vi /etc/sysctl.conf`

with the content:

```ini
net.ipv4.ip_forward = 1
```

- `firewall-cmd --permanent --zone=public --set-target=ACCEPT`
- `firewall-cmd --add-port=1723/tcp --permanent`
- `systemctl enable --now openvswitch.service network.service os-autoinst-openvswitch.service`
- `ovs-vsctl add-br br0`
- `vi /etc/openqa/workers.ini`

with the content:

```ini
[global]
WORKER_CLASS = qemu_x86_64,tap
```

- `setcap CAP_NET_ADMIN=ep /usr/bin/qemu-system-x86_64`
- `reboot`

### Worker Host configuration

- `dnf install os-autoinst-openvswitch tunctl network-scripts`
- `vi /etc/sysconfig/os-autoinst-openvswitch`

with the content:

```ini
OS_AUTOINST_BRIDGE_LOCAL_IP=172.16.2.<2 + counting-number>
OS_AUTOINST_BRIDGE_REWRITE_TARGET=172.17.0.0
```

- `vi /etc/sysconfig/network-scripts/ifcfg-br0`

with the content:

```ini
DEVICETYPE='ovs'
TYPE='OVSBridge'
BOOTPROTO='static'
IPADDR='172.16.2.<2 + counting-number>'
NETMASK='255.254.0.0'
DEVICE=br0
STP=off
ONBOOT='yes'
NAME='br0'
HOTPLUG='no'
```

For every openqa-worker 1 tap interface (worker@1 = tap0, worker@2 = tap1, ...)

- `vi /etc/sysconfig/network-scripts/ifcfg-tap0`

with the content:

```ini
DEVICETYPE='ovs'
TYPE='OVSPort'
OVS_BRIDGE='br0'
DEVICE='tap0'
ONBOOT='yes'
BOOTPROTO='none'
HOTPLUG='no'
```

- `vi /sbin/ifup-pre-local`

In the last line, configure the GRE tunnel which was created on the master by using the same counting-number as on the master

with the content:

```sh
#!/bin/sh

if=$(echo "$1" | sed -e 's,ifcfg-,,')
iftype=$(echo "$if" | sed -e 's,[0-9]\+$,,')

# if the interface being brought up is tap[n], create
# the tap device first
if [ "$iftype" == "tap" ]; then
    tunctl -u _openqa-worker -p -t "$if"
fi

# if the interface being brough up is br0, create
# the gre tunnels
if [ "$if" == "br0" ]; then
    ovs-vsctl set bridge br0 stp_enable=true
    ovs-vsctl --may-exist add-port br0 gre<counting-number> -- set interface gre<counting-number> type=gre options:remote_ip=<ip-of-external-worker-host>
fi
```

- `chmod ug+x /sbin/ifup-pre-local`
- `firewall-cmd --permanent --zone=internal --add-interface=br0`
- `firewall-cmd --add-port=1723/tcp --permanent`
- `systemctl enable --now openvswitch.service network.service os-autoinst-openvswitch.service`
- `ovs-vsctl add-br br0`
- `vi /etc/openqa/workers.ini`

with the content:

```ini
[global]
WORKER_CLASS = qemu_x86_64,tap
```

- `setcap CAP_NET_ADMIN=ep /usr/bin/qemu-system-x86_64`
- `reboot`
