# Docker

## Network bridge connected to wrong FirewallD zone after online network changes

If you i.e. change the DNS name of a physical NIC with NetworkManager,
it might happen that a docker bridge suddenly is connected to the default zone:

```bash
nmtui <change dns>
systemctl restart NetworkManager
```

After this the bridge is wrong, and the NAT to the outside world does not work anymore,
to fix that, rehook the bridge to the docker zone again, and restart docker:

```bash
firewall-cmd --remove-interface=brxyz-xyz
firewall-cmd --zone=docker --add-interface=brxyz-xyz
systemctl restart docker
```

Then it should work again.

The other way to fix this, is to only change the DNS entry, and then reboot the whole system.
