# PowerDNS application setups

This setup of PowerDNS brings up a full 2-node HA cluster one of them being primary, and will include `pdns` itself, but also `pdns-recursor` and `dnsdist`

It's meant to be setup with Rocky Linux 10, but it will generally work on many distributions, whith small tweaks to the documentation.

Documentation:  
[PowerDNS Authorative Server](https://doc.powerdns.com/authoritative/index.html)
[PowerDNS Recursor](https://doc.powerdns.com/recursor/index.html)
[PowerDNS dnsdist](https://www.dnsdist.org/index.html)

## All cluster nodes - 1

- Setup a seperate IP for the authorative server and the frontend/dnsdist (2 per node)
- There will also be a floating VIP being deployed, so prepare this one too
- `dnf install setroubleshoot-server epel-release`
- `crb enable`
- `dnf config-manager --add-repo https://repo.powerdns.com/repo-files/el-auth-51.repo`
- `dnf config-manager --add-repo https://repo.powerdns.com/repo-files/el-rec-54.repo`
- `dnf config-manager --add-repo https://repo.powerdns.com/repo-files/el-dnsdist-21.repo`
- `dnf install postgresql-server`
- `postgresql-setup --initdb`
- Make sure authentication for PowerDNS is set to `scram-sha-256` in the `pg_hba.conf` file
- `systemctl enable --now postgresql`
- `dnf install pdns pdns-backend-postgresql pdns-recursor dnsdist`
- Create and populate the database:

```bash
su - postgres
psql
create database pdns;
\q
psql pdns < /usr/share/doc/pdns-backend-postgresql/schema.pgsql.sql
psql pdns
create user pdns with password '<secure-password>';
grant all on database pdns to pdns;
grant all on all tables in schema public to pdns;
grant all on all sequences in schema public to pdns;
\q
exit
```

- Change/Add the following parameters in `/etc/pdns/pdns.conf`:

```conf
api=yes
api-key=<secure-api-key>
default-soa-content=<fqdn-of-primary> <hostname-of-hostmaster-record>.@ 0 10800 3600 604800 3600
default-soa-edit=INCEPTION-INCREMENT
launch=gpgsql
local-address=<ip-of-auth-server>
webserver-address=<ip-of-auth-server>
webserver-allow-from=127.0.0.1,::1,<powerdns-admin-ip>
gpgsql-host=127.0.0.1
gpgsql-dbname=pdns
gpgsql-user=pdns
gpgsql-password=<secure-password>
gpgsql-dnssec=yes
```

- `chown pdns:pdns /etc/pdns/pdns.conf`

## PowerDNS Authorative Server - primary

- Set `primary=true` additionally in the `/etc/pdns/pdns.conf` file
- `systemctl enable --now pdns`
- `pdnsutil create-zone <domain>`
- `pdnsutil set-kind <domain> primary`
- `pdnsutil set-meta <domain> API-RECTIFY 1`
- `pdnsutil set-meta <domain> SOA-EDIT-API DEFAULT`
- Edit the zone content with `EDITOR=vi pdnsutil edit-zone <domain>`:

```text
; Warning - every name in this file is ABSOLUTE!
$ORIGIN .
<domain>            3600    IN      SOA     <fqdn-of-primary> <fqdn-of-hostmaster-record> 0 10800 3600 604800 3600
<domain>            60      IN      NS      <fqdn-of-primary>
<domain>            60      IN      NS      <fqdn-of-secondary>
<fqdn-of-primary>   60      IN      A       <ip-of-primary>
<fqdn-of-secondary> 60      IN      A       <ip-of-primary>
```

- `pdnsutil secure-zone <domain>`
- `pdnsutil generate-tsig-key <keyname> <hmac-md5|hmac-shaX>`
- `pdnsutil activate-tsig-key <domain-to-sign> <keyname> primary`
- `pdnsutil show-zone <domain>` to get the DS keys for later

## PowerDNS Authorative Server - secondary

- Set `secondary=true` additionally in the `/etc/pdns/pdns.conf` file
- `systemctl enable --now pdns`
- `pdnsutil create-secondary-zone <domain> <ip-of-primary>`
- `pdnsutil import-tsig-key <keyname> <hmac-md5|hmac-shaX> '<base64-encoded-secret>'`
- `pdnsutil activate-tsig-key <domain-to-sign> <keyname> secondary`
- Make a change on the primary or run `pdns_control notify <domain>` on the primary
- `firewall-cmd --add-port=8081/tcp`
- `firewall-cmd --add-port=8081/tcp --permanent`

## All cluster nodes - 2

- Setup the Recursor configuration as follows:

```yaml
dnssec:
  validation: log-fail
  trustanchors:
    - name: <domain>
      dsrecords:
        - <sha256-ds-string>
        - <sha384-ds-string>
recursor:
  include_dir: /etc/pdns-recursor/recursor.d
  setuid: pdns-recursor
  setgid: pdns-recursor
  forward_zones:
    - zone: <domain>
      forwarders:
        - <ip-of-auth-server>:53
incoming:
  listen:
    - 127.0.0.1:5300
    - '[::1]:5353'
```

- `systemctl enable --now pdns-recursor`
- Setup the dnsdist configuration as follows:

```yaml
acl:
  - 0.0.0.0/0

binds:
  - listen_address: "<ip-of-recursor>:53"
    protocol: Do53
  - listen_address: "<ip-of-vip>:53"
    protocol: Do53
#  - listen_address: "<ip-of-recursor>:443"
#    protocol: DoH
#    tls:
#      certificates:
#        - certificate: /etc/dnsdist/cert.pem
#          key: /etc/dnsdist/privkey.pem
#  - listen_address: "<ip-of-vip>:443"
#    protocol: DoH
#    tls:
#      certificates:
#        - certificate: /etc/dnsdist/cert.pem
#          key: /etc/dnsdist/privkey.pem
#  - listen_address: "<ip-of-recursor>:443"
#    protocol: DoH3
#    tls:
#      certificates:
#        - certificate: /etc/dnsdist/cert.pem
#          key: /etc/dnsdist/privkey.pem
#  - listen_address: "<ip-of-vip>:443"
#    protocol: DoH3
#    tls:
#      certificates:
#        - certificate: /etc/dnsdist/cert.pem
#          key: /etc/dnsdist/privkey.pem

backends:
  - address: "127.0.0.1:5300"
    protocol: Do53
    pools:
      - default

query_rules:
  - name: "default rule"
    selector:
      type: All
    action:
      type: Pool
      pool_name: default
```

- `systemctl enable --now dnsdist`
- `firewall-cmd --add-service={dns,https}`
- `firewall-cmd --add-service={dns,https} --permanent`

## Setup Pacemaker for floating VIP

- `dnf config-manager --enable highavailability`
- `dnf install which pacemaker pcs sbd`
- `echo softdog > /etc/modules-load.d/softdog.conf`
- `modprobe softdog`
- Set `SBD_DELAY_START=yes` in the `/etc/sysconfig/sbd` file
- `firewall-cmd --add-service=high-availability`
- `firewall-cmd --add-service=high-availability --permanent`
- `systemctl enable --now pcsd sbd`
- `passwd hacluster` and set a secure password
- `pcs host auth <fqdn-of-primary> <fqdn-of-secondary> <fqdn-of-secondary-n+1` and enter the used password
- `pcs cluster setup pdns --start <fqdn-of-primary> <fqdn-of-secondary> <fqdn-of-secondary-n+1`
- `pcs cluster enable --all`
- `pcs property set stonith-enabled=true`
- `pcs property set stonith-watchdog-timeout=20`
- `mkdir /usr/lib/ocf/resource.d/custom`
- Create a custom resource agent for pacemaker and safe to `/usr/lib/ocf/resource.d/custom/pdns_check`:

```bash
#!/bin/bash

. /usr/lib/ocf/lib/heartbeat/ocf-shellfuncs

meta_data() {
cat <<END
<?xml version="1.0"?>
<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
<resource-agent name="pdns_check">
<version>1.0</version>

<longdesc lang="en">
Checks PowerDNS availability via TCP and DNS query.
</longdesc>

<shortdesc lang="en">PowerDNS health check</shortdesc>

<parameters>
</parameters>

<actions>
<action name="start" timeout="5s" />
<action name="stop" timeout="5s" />
<action name="monitor" timeout="5s" interval="10s" />
<action name="meta-data" timeout="5s" />
</actions>

</resource-agent>
END
}


check() {
    dig @<ip-of-auth-server> <fqdn-of-primary> +short +time=2 +tries=1 >/dev/null 2>&1 || return 1
    dig @127.0.0.1 -p 5353 <fqdn-of-primary> +short +time=2 +tries=1 >/dev/null 2>&1 || return 1
    dig @<ip-of-recursor> <fqdn-of-primary> +short +time=2 +tries=1 >/dev/null 2>&1 || return 1

    return 0
}

case "$1" in
    meta-data)
        meta_data
        exit 0
        ;;
    start|stop)
        exit 0
        ;;
    monitor)
        check || exit $OCF_ERR_GENERIC
        exit 0
        ;;
    status)
        check
        ;;
    *)
        exit $OCF_ERR_UNIMPLEMENTED
        ;;
esac
```

- `pcs resource create pdns_vip IPaddr2 ip=<ip-of-vip> cidr_netmask=24 op monitor interval=30s`
- `pcs resource create pdns_check ocf:custom:pdns_check op monitor interval=10s timeout=5s meta migration-threshold=1 failure-timeout=30s`
- `pcs constraint colocation add pdns_vip with pdns_check INFINITY`
- `pcs constraint order pdns_check then pdns_vip`
- `pcs constraint location pdns_vip prefers <fqdn-of-primary>=100`

## PowerDNS-Admin

Documentation source: [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/blob/master/README.md#running-powerdns-admin) and [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/wiki/Running-PowerDNS-Admin-with-Systemd,-Gunicorn--and--Nginx)

- `dnf install podman`
- `mkdir /etc/containers/systemd/powerdns-admin`
- Create the following `/etc/containers/systemd/powerdns-admin/powerdns-admin.pod` file:

```ini
[Pod]
PublishPort=127.0.0.1:9191:80

[Install]
WantedBy=default.target
```

- Create the following `/etc/containers/systemd/powerdns-admin/pda-legacy.container` file:

```ini
[Unit]
Description=PowerDNS Admin

[Container]
AutoUpdate=registry
Pod=powerdns-admin.pod
Image=docker.io/powerdnsadmin/pda-legacy:latest
Volume=pda-data:/data
Environment=SECRET_KEY=<secret-key>

[Service]
TimeoutStartSec=900
```

- `systemctl daemon-reload`
- `systemctl start powerdns-admin-pod`
- Add nginx configuration in `/etc/nginx/conf.d/powerdns-admin.conf`:

```conf
server {
  listen <ip-of-auth-server>:80;
  server_name <fqdn-of-primary>;

  return 301 https://$host$request_uri;
}

server {
  listen <ip-of-auth-server>:443 ssl;
  http2 on;
  server_name <fqdn-of-primary>;
  index index.html index.htm

  proxy_read_timeout 720s;
  proxy_connect_timeout 720s;
  proxy_send_timeout 720s;
  proxy_redirect off;

  client_max_body_size 50m;
  client_body_buffer_size 128k;

  # Proxy headers
  proxy_set_header Host $host;
  proxy_set_header X-Scheme $scheme;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_headers_hash_bucket_size 64;

  # SSL parameters
  ssl_certificate /path/to/cert.pem; # managed by Certbot
  ssl_certificate_key /path/to/privkey.pem; # managed by Certbot
  ssl_protocols TLSv1.2 TLSv1.3;

  # log files
  access_log /var/log/nginx/pda-access.log;
  error_log /var/log/nginx/pda-error.log;

  location / {
    proxy_pass              http://127.0.0.1:9191;
    proxy_read_timeout      120;
    proxy_connect_timeout   120;
  }

}
```

- `systemctl enable --now nginx`
- `firewall-cmd --add-service={http,https}`
- `firewall-cmd --add-service={http,https} --permanent`
- Go to web-frontend and register initial user
- Configure the connection to the powerdns system using the URI (port 8081), secret and version
- You might have to update the webserver allow list again, depending on your setup
