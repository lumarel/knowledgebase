# PowerDNS application setups

## pdns - cluster - primary

[Documentation source](https://doc.powerdns.com/authoritative/)

- `dnf install dnf-utils setroubleshoot-server epel-release`
- `dnf config-manager --enable powertools`
- `dnf config-manager --add-repo https://repo.powerdns.com/repo-files/el-auth-46.repo`
- `dnf module enable postgresql:13`
- `dnf install postgresql-server`
- `postgresql-setup --initdb`
- Change the authentication type in `/var/lib/pgsql/data/pg_hba.conf` from `ident` to `md5` for the local connections
- `systemctl enable --now postgresql`
- `dnf install pdns pdns-backend-postgresql`
- Create and populate the database:

```bash
su - postgres
psql
create database pdns;
\q
psql pdns < /usr/share/doc/pdns-backend-postgresql/schema.pgsql.sql
psql
\c pdns
create user pdns with password '<secure-password>';
grant all on database pdns to pdns;
grant all on all tables in schema public to pdns;
grant all on all sequences in schema public to pdns;
\c postgres
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
primary=yes
webserver-address=0.0.0.0
webserver-allow-from=<powerdns-admin-ip>
gpgsql-host=127.0.0.1
gpgsql-dbname=pdns
gpgsql-user=pdns
gpgsql-password=<secure-password>
gpgsql-dnssec=yes
```

- `chown pdns:pdns /etc/pdns/pdns.conf`
- `systemctl enable --now pdns`
- `firewall-cmd --add-service=dns`
- `firewall-cmd --add-service=dns --permanent`
- `firewall-cmd --add-port=8081/tcp`
- `firewall-cmd --add-port=8081/tcp --permanent`
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

## pdns - cluster - secondary

[Documentation source](https://doc.powerdns.com/authoritative/)

- `dnf install dnf-utils setroubleshoot-server epel-release`
- `dnf config-manager --enable powertools`
- `dnf config-manager --add-repo https://repo.powerdns.com/repo-files/el-auth-46.repo`
- `dnf module enable postgresql:13`
- `dnf install postgresql-server`
- `postgresql-setup --initdb`
- Change the authentication type in `/var/lib/pgsql/data/pg_hba.conf` from `ident` to `md5` for the local connections
- `systemctl enable --now postgresql`
- `dnf install pdns pdns-backend-postgresql`
- Create and populate the database:

```bash
su - postgres
psql
create database pdns;
\q
psql pdns < /usr/share/doc/pdns-backend-postgresql/schema.pgsql.sql
psql
\c pdns
create user pdns with password '<secure-password>';
grant all on database pdns to pdns;
grant all on all tables in schema public to pdns;
grant all on all sequences in schema public to pdns;
\c postgres
\q
exit
```

- Change/Add the following parameters in `/etc/pdns/pdns.conf`:

```conf
api=yes
api-key=<secure-api-key>
default-soa-content=<fqdn-of-secondary> <hostname-of-hostmaster-record>.@ 0 10800 3600 604800 3600
default-soa-edit=INCEPTION-INCREMENT
launch=gpgsql
secondary=yes
webserver-address=0.0.0.0 # (optional)
webserver-allow-from=<powerdns-admin-ip> # (optional)
gpgsql-host=127.0.0.1
gpgsql-dbname=pdns
gpgsql-user=pdns
gpgsql-password=<secure-password>
gpgsql-dnssec=yes
```

- `chown pdns:pdns /etc/pdns/pdns.conf`
- `systemctl enable --now pdns`
- `firewall-cmd --add-service=dns`
- `firewall-cmd --add-service=dns --permanent`
- `firewall-cmd --add-port=8081/tcp`
- `firewall-cmd --add-port=8081/tcp --permanent`
- `pdnsutil create-secondary-zone <domain> <ip-of-primary>`
- `pdnsutil import-tsig-key <keyname> <hmac-md5|hmac-shaX> '<base64-encoded-secret>'`
- `pdnsutil activate-tsig-key <domain-to-sign> <keyname> secondary`
- Make a change on the primary or run `pdns_control notify <domain>` on the primary

## PowerDNS-Admin

Documentation source: [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/wiki/Using-PowerDNS-Admin-with-PostgreSQL), [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/wiki/Running-PowerDNS-Admin-on-Fedora-30) and [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/wiki/Running-PowerDNS-Admin-with-Systemd,-Gunicorn--and--Nginx)

- `dnf install dnf-utils setroubleshoot-server`
- `dnf config-manager --enable powertools`
- `dnf module enable postgresql:13`
- `dnf module enable nginx:1.20`
- `dnf install postgresql-server python3-psycopg2 nginx python3-devel python3-pip openldap-devel xmlsec1-devel xmlsec1-openssl libtool-ltdl-devel gcc gc make npm git mariadb-connector-c-devel libpq-devel`
- `postgresql-setup --initdb`
- Change the authentication type in `/var/lib/pgsql/data/pg_hba.conf` from `ident` to `md5` for the local connections
- `systemctl enable --now postgresql`
- Create and populate the database:

```bash
su - postgres
create database powerdnsadmindb;
create user powerdnsadmin with encrypted password '<secure-password>';
grant all privileges on database powerdnsadmindb to powerdnsadmin;
\q
exit
```

- `python3 -m pip install -U pip`
- `python3 -m pip install -U virtualenv`
- `npm install yarn -g`
- `mkdir -p /opt/web`
- `cd /opt/web`
- `git clone https://github.com/PowerDNS-Admin/PowerDNS-Admin.git powerdns-admin`
- `cd /opt/web/powerdns-admin`
- `virtualenv -p python3 flask`
- `. ./flask/bin/activate`
- `python3 -m pip install python-dotenv`
- `python3 -m pip install psycopg2`
- `python3 -m pip install -r requirements.txt`
- Change the DB user, password, name, to the correct one, also change the databasetype from mysql to postgresql in the `SQLALCHEMY_DATABASE_URI` variable in `/opt/web/powerdns-admin/powerdnsadmin/default_config.py`
- `export FLASK_APP=powerdnsadmin/__init__.py`
- `flask db upgrade`
- `yarn install --pure-lockfile`
- `flask assets build`
- `cp /opt/web/powerdns-admin/configs/development.py /opt/web/powerdns-admin/configs/production.py`
- Change the same parameters as before in `/opt/web/powerdns-admin/configs/production.py`, uncomment the whole `SQLALCHEMY_DATABASE_URI` variable, comment the SQLite variable, change the `SECRET_KEY` variable and uncomment the urllib import
- `useradd -r powerdnsadmin`
- `chown -R powerdnsadmin:powerdnsadmin /opt/web/powerdns-admin`
- Add the systemd service in `/etc/systemd/system/powerdns-admin.service`:

```ini
[Unit]
Description=PowerDNS-Admin
Requires=powerdns-admin.socket
After=network.target

[Service]
PIDFile=/run/powerdns-admin/pid
User=powerdnsadmin
Group=powerdnsadmin
WorkingDirectory=/opt/web/powerdns-admin
ExecStart=/usr/local/bin/gunicorn --pid /run/powerdns-admin/pid --bind unix:/run/powerdns-admin/socket 'powerdnsadmin:create_app()'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

- And change it to the following with `systemctl edit powerdns-admin.service`:

```ini
[Service]
Environment="FLASK_CONF=../configs/production.py"
```

- Add the systemd socket in `/etc/systemd/system/powerdns-admin.socket`:

```ini
[Unit]
Description=PowerDNS-Admin socket

[Socket]
ListenStream=/run/powerdns-admin/socket

[Install]
WantedBy=sockets.target
```

- Add the tmpfile config in `/etc/tmpfiles.d/powerdns-admin.conf`:

```text
d /run/powerdns-admin 0755 powerdnsadmin powerdnsadmin -
```

- Reload systemd with `systemctl daemon-reload`
- Start the systemd socket with `systemctl enable --now powerdns-admin.socket`
- Start the systemd service with `systemctl enable --now powerdns-admin.service`
- Add nginx configuration in `/etc/nginx/conf.d/powerdns-admin.conf`:

```conf
server {
    listen                  80;
    listen                  [::]:80;
    server_name             <fqdn>;
    return 301 https://$http_host$request_uri;
}

server {
    listen                  443 ssl http2;
    listen                  [::]:443 ssl http2;
    server_name             <fqdn>;
    index                   index.html index.htm;
    error_log               /var/log/nginx/powerdns-admin.error.log;
    access_log              /var/log/nginx/powerdns-admin.access.log;

    ssl_certificate                 <path_to_your_fullchain_or_cert>;
    ssl_certificate_key             <path_to_your_key>;
    ssl_protocols                   TLSv1.2 TLSv1.3;
    ssl_session_cache               shared:SSL:10m;

    client_max_body_size            50m;
    client_body_buffer_size         128k;
    proxy_redirect                  off;
    proxy_connect_timeout           720s;
    proxy_send_timeout              720s;
    proxy_read_timeout              720s;
    proxy_buffers                   32 4k;
    proxy_buffer_size               8k;
    proxy_set_header                Host $http_host;
    proxy_set_header                X-Scheme $scheme;
    proxy_set_header                X-Real-IP $remote_addr;
    proxy_set_header                X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_headers_hash_bucket_size  64;

    location ~ ^/static/  {
        include         mime.types;
        root            /opt/web/powerdns-admin/powerdnsadmin;
        location        ~* \.(jpg|jpeg|png|gif)$ { expires 365d; }
        location        ~* ^.+.(css|js)$ { expires 7d; }
    }

    location ~ ^/upload/  {
        include         mime.types;
        root            /opt/web/powerdns-admin;
        location        ~* \.(jpg|jpeg|png|gif)$ { expires 365d; }
        location        ~* ^.+.(css|js)$ { expires 7d; }
    }

    location / {
        proxy_pass              http://unix:/run/powerdns-admin/socket;
        proxy_read_timeout      120;
        proxy_connect_timeout   120;
        proxy_redirect          http:// $scheme://;
    }
}
```

- `systemctl enable --now nginx`
- `firewall-cmd --add-service={http,https}`
- `firewall-cmd --add-service={http,https} --permanent`
- Go to web-frontend and register initial user
- Configure the connection to the powerdns system using the URI (port 8081), secret and version

## Update PowerDNS-Admin

- `systemctl stop powerdns-admin.service`
- `systemctl stop powerdns-admin.socket`
- `npm update yarn -g`
- `cd /opt/web/powerdns-admin`
- If not needed anymore restore the default file in the `powerdnsadmin` directory with `git restore powerdnsadmin/default_config.py`
- `git fetch`
- `git pull`
- `. ./flask/bin/activate`
- `python3 -m pip install -U python-dotenv`
- `python3 -m pip install -U psycopg2`
- `python3 -m pip install -U -r requirements.txt`
- `export FLASK_APP=powerdnsadmin/__init__.py`
- `export FLASK_CONF=../configs/production.py`
- `flask db upgrade`
- `yarn upgrade --pure-lockfile`
- `flask assets build`
- `chown -R powerdnsadmin:powerdnsadmin /opt/web/powerdns-admin`
- `systemctl start powerdns-admin.socket`
- `systemctl start powerdns-admin.service`

## PowerDNS-Admin - easy-mode

Documentation source: [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/blob/master/README.md#running-powerdns-admin) and [here](https://github.com/PowerDNS-Admin/PowerDNS-Admin/wiki/Running-PowerDNS-Admin-with-Systemd,-Gunicorn--and--Nginx)

- Install docker
- `docker run -d -e SECRET_KEY='<secure-password>' -v pda-data:/data -p 9191:80 ngoduykhanh/powerdns-admin:latest`
- Add nginx configuration in `/etc/nginx/conf.d/powerdns-admin.conf`:

```conf
server {
    listen                  80;
    listen                  [::]:80;
    server_name             <fqdn>;
    return 301 https://$http_host$request_uri;
}

server {
    listen                  443 ssl http2;
    listen                  [::]:443 ssl http2;
    server_name             <fqdn>;
    index                   index.html index.htm;
    error_log               /var/log/nginx/powerdns-admin.error.log;
    access_log              /var/log/nginx/powerdns-admin.access.log;

    ssl_certificate                 <path_to_your_fullchain_or_cert>;
    ssl_certificate_key             <path_to_your_key>;
    ssl_protocols                   TLSv1.2 TLSv1.3;
    ssl_session_cache               shared:SSL:10m;

    client_max_body_size            50m;
    client_body_buffer_size         128k;
    proxy_redirect                  off;
    proxy_connect_timeout           720s;
    proxy_send_timeout              720s;
    proxy_read_timeout              720s;
    proxy_buffers                   32 4k;
    proxy_buffer_size               8k;
    proxy_set_header                Host $http_host;
    proxy_set_header                X-Scheme $scheme;
    proxy_set_header                X-Real-IP $remote_addr;
    proxy_set_header                X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_headers_hash_bucket_size  64;

    location ~ ^/static/  {
        include         mime.types;
        root            /opt/web/powerdns-admin/powerdnsadmin;
        location        ~* \.(jpg|jpeg|png|gif)$ { expires 365d; }
        location        ~* ^.+.(css|js)$ { expires 7d; }
    }

    location ~ ^/upload/  {
        include         mime.types;
        root            /opt/web/powerdns-admin;
        location        ~* \.(jpg|jpeg|png|gif)$ { expires 365d; }
        location        ~* ^.+.(css|js)$ { expires 7d; }
    }

    location / {
        proxy_pass              http://unix:/run/powerdns-admin/socket;
        proxy_read_timeout      120;
        proxy_connect_timeout   120;
        proxy_redirect          http:// $scheme://;
    }
}
```

- `systemctl enable --now nginx`
- `firewall-cmd --add-service={http,https}`
- `firewall-cmd --add-service={http,https} --permanent`
- Go to web-frontend and register initial user
- Configure the connection to the powerdns system using the URI (port 8081), secret and version
