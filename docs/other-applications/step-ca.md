# Step CA setups

## Sub CA usage as ACME endpoint

[Basic Install](https://smallstep.com/docs/step-ca/installation)

[Basic Configuration](https://smallstep.com/docs/step-ca/configuration)

[Setup Intermediate CA](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca#the-secure-way)

[ACME Basics](https://smallstep.com/docs/step-ca/acme-basics)

[ACME Provisioner](https://smallstep.com/docs/step-ca/provisioners#acme)

[ACME Clients](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients)

- Setup smallstep repository:

```bash
cat <<EOF > /etc/yum.repos.d/smallstep.repo
[smallstep]
name=Smallstep
baseurl=https://packages.smallstep.com/stable/fedora/
enabled=1
repo_gpgcheck=0
gpgcheck=1
gpgkey=https://packages.smallstep.com/keys/smallstep-0x889B19391F774443.gpg
EOF
```

- Install `step-ca` and `step` with `dnf install step-ca step-cli`
- Init the step-ca with `step ca init`
  - Use `Standalone`
  - Name the CA i.e. `subca01`
  - Add all the FQDNs the CA should listen on i.e. `subca01.example.com`
  - To bind on 443 on every IP use `:443`
  - Define the mail address of the first provisioner i.e. `subca01@example.com`
  - Let it generate a password with `[enter]` and a second time to save (this password is not used and will be deleted soon again)
- Delete the root_ca_key with `rm ~/.step/secrets/*`
- Change the `root_ca.crt` to the public certificate of your root_ca with `vi ~/.step/certs/root_ca.crt`
- Add a safe enough password to the `key.pass` file with `vi ~/.step/key.pass` (you can use the generated cert from the init)
- Create the CSR for the intermediate_ca:

```bash
step certificate create <subject-of-cert> intermediate.csr ~/.step/secrets/intermediate_ca_key --csr [--san subca01.example.com [--san subca01.example.com]] --password-file ~/.step/key.pass
```

- Copy the CSR to the root_ca and sign it
  - For ADCS it looks like that for signing:

    ```batch
    certreq -submit -attrib "CertificateTemplate:SubCA" intermediate.csr intermediate.crt
    ```

- Copy the signed public cert back to the machine with `vi ~/.step/certs/intermediate_ca.crt`
- Add the ACME endpoint with `step ca provisioner add acme --type ACME`
- Remove the JWK endpoint with `step ca provisioner remove subca01@example.com --type JWK` (with the mail address from the init)
- Move the whole configuration to the `/etc` with `mv ~/.step /etc/step-ca`
- Create a folder in `/var/lib` with `mkdir /var/lib/step-ca`
- Move the database to that folder with `mv /etc/step-ca/db /var/lib/step-ca`
- Change the paths from `~/.step` to `/etc/step-ca` in the two files `config/defaults.json` and `config/ca.json`
- And change the `db.dataSource` path to `/var/lib/step-ca/db` in the `config/ca.json` file
- Create an account for the service with `useradd --system --home /etc/step-ca --shell /bin/false step`
- Fix the permissions for both directories:

```bash
chown -R step:step /etc/step-ca
restorecon -R /etc/step-ca
chmod u=r,g=,o= /etc/step-ca/key.pass
chown -R step:step /var/lib/step-ca
restorecon -R /var/lib/step-ca
```

- Create the systemd config with `vi /etc/systemd/system/step-ca.service` and the content:

```ini
[Unit]
Description=step-ca service
Documentation=https://smallstep.com/docs/step-ca
Documentation=https://smallstep.com/docs/step-ca/certificate-authority-server-production
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=30
StartLimitBurst=3
ConditionFileNotEmpty=/etc/step-ca/config/ca.json
ConditionFileNotEmpty=/etc/step-ca/key.pass

[Service]
Type=simple
User=step
Group=step
Environment=STEPPATH=/etc/step-ca
WorkingDirectory=/etc/step-ca
ExecStart=/usr/local/bin/step-ca config/ca.json --password-file key.pass
ExecReload=/bin/kill --signal HUP $MAINPID
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
StartLimitInterval=30
StartLimitBurst=3

; Process capabilities & privileges
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
SecureBits=keep-caps
NoNewPrivileges=yes

; Sandboxing
ProtectSystem=full
ProtectHome=true
RestrictNamespaces=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
PrivateTmp=true
PrivateDevices=true
ProtectClock=true
ProtectControlGroups=true
ProtectKernelTunables=true
ProtectKernelLogs=true
ProtectKernelModules=true
LockPersonality=true
RestrictSUIDSGID=true
RemoveIPC=true
RestrictRealtime=true
SystemCallFilter=@system-service
SystemCallArchitectures=native
MemoryDenyWriteExecute=true
ReadWriteDirectories=/var/lib/step-ca/db

[Install]
WantedBy=multi-user.target
```

- Reload systemd with `systemctl daemon-reload`
- Start the systemd service with `systemctl enable --now step-ca.service`
- And finally open the firewall:

```bash
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

- To use some `step` commands after the move you will need to add some variables to your profile (like the `~/.bashrc`):

```bash
export STEPPATH=/etc/step-ca
export STEP_CA_URL=https://subca01.example.com
export STEP_ROOT=/etc/step-ca/certs/root_ca.crt
```

## Production considerations

### PostgreSQL as database

- Before installing setup a PostgreSQL database:

```bash
dnf install postgresql-server
postgresql-setup initdb
```

- Change the 2 `host all all` lines from ident to `md5` (unfortunately `scram-sha-256` does not work)

```bash
systemctl enable --now postgresql
sudo -u postgres psql
```

- And inside psql run:

```sql
create database stepca;
create user stepca with encrypted password '<strong-password>';
grant all privileges on database stepca to stepca;
```

- After the init, change the db settings in `~/.step/config/ca.json`:

```json
"db": {
  "type": "postgresql",
  "dataSource": "postgresql://stepca:<strong-password>@127.0.0.1:5432/",
  "database": "stepca"
}
```

### Active revocation

[Enable Active Revocation](https://smallstep.com/docs/step-ca/certificate-authority-server-production/#consider-active-revocation)

For all CRLs provided in the certificates, the `crlDistributionPoints` variable has to exactly match the CDP variable in the CRL!  
By default the `idpURL` is `https://<ca-fqdn>/1.0/crl`.

#### Intermediate CA certificates

- Before creating the CSR create a template file in `~/.step/templates/intermediate.tpl` with the following content:

```json
{
        "subject": {{ toJson .Subject }},
        "keyUsage": ["certSign", "crlSign"],
        "basicConstraints": {
                "isCA": true,
                "maxPathLen": 0
        },
        "issuingCertificateURL": "http://crl.example.com/rootca.crt",
        "crlDistributionPoints": ["http://subca01.example.com/1.0/crl"]
}
```

- After that run the this command instead of the other CSR create:

```bash
step certificate create <subject-of-cert> intermediate.csr ~/.step/secrets/intermediate_ca_key --csr [--san <subject-of-cert>.example.com [--san subca01.example.com]] --password-file ~/.step/key.pass --template intermediate.tpl
```

##### Client certificates

For custom client certificates for either all or specific providers setup a template for each case.

- Create a template like `templates/server.tpl`:

```json
{
  "subject": {{ toJson .Subject }},
  "sans": {{ toJson .SANs }},
{{- if typeIs "*rsa.PublicKey" .Insecure.CR.PublicKey }}
  "keyUsage": ["keyEncipherment", "digitalSignature"],
{{- else }}
  "keyUsage": ["digitalSignature"],
{{- end }}
  "extKeyUsage": ["serverAuth", "clientAuth"],
  "issuingCertificateURL": "http://crl.example.com/subca01.crt",
  "crlDistributionPoints": ["http://subca01.example.com/1.0/crl"]
}
```

- And then reference the templates relative path in the `ca.json` config file

## Links for operation

- [badger database cleanup](https://github.com/smallstep/certificates/issues/473)
- [logging/certificate reading from database](https://github.com/smallstep/certificates/issues/239)
- [CRL setups with reverse proxy inbetween](https://github.com/smallstep/certificates/issues/2150)
