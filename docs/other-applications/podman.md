# Podman

## Socket permissions for Zabbix

To monitor Podman with Zabbix it's necessary to edit the socket permissions.

Add podman.socket override (`systemctl edit podman.socket`):

```ini
[Socket]
SocketGroup=podman
```

(very important!) Set an override for tmpfiles:

```bash
cat <<EOF | tee /etc/tmpfiles.d/podman.conf
D! /run/podman 0750 root podman

EOF
```

Enable socket with `systemctl enable --now podman.socket`.

Edit the Zabbix Agent2 Docker plugin configuration to use the podman socket (also mitigatable by installing `podman-docker`):

```ini
Plugins.Docker.Endpoint=unix:///var/run/podman/podman.sock
```
