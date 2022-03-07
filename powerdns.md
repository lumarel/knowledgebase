# PowerDNS application setups

## pdns - cluster - primary

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
pqsl pdns < /usr/share/doc/pdns-backend-postgresql/schema.pgsql.sql
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
default-soa-content=<hostname-of-primary> <hostname-of-hostmaster-record>.@ 0 10800 3600 604800 3600
launch=gpgsql
local-address=127.0.0.1, ::1
local-port=5300
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

## PowerDNS-Admin

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
- `git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git powerdns-admin`
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
- Configure nginx to proxy to the socket TBD
- `firewall-cmd --add-service={http,https}`
- `firewall-cmd --add-service={http,https} --permanent`
- Go to web-frontend and register initial user
