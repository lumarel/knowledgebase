# Step CA setups

## Sub CA usage as ACME endpoint

[Basic Install](https://smallstep.com/docs/step-ca/installation)

[Basic Configuration](https://smallstep.com/docs/step-ca/configuration)

[Setup Intermediate CA](https://smallstep.com/docs/tutorials/intermediate-ca-new-ca#the-secure-way)

[ACME Basics](https://smallstep.com/docs/step-ca/acme-basics)

[ACME Provisioner](https://smallstep.com/docs/step-ca/provisioners#acme)

[ACME Clients](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients)

- Download the current version of `step-ca` and `step` from the Github releases

```bash
curl -O https://dl.step.sm/gh-release/certificates/docs-ca-install/v0.19.0/step-ca_linux_0.19.0_amd64.tar.gz
tar -xzvf step-ca_linux_0.19.0_amd64.tar.gz
mv step-ca_0.19.0/bin/step-ca /usr/local/bin
chown root:root /usr/local/bin/step-ca
restorecon /usr/local/bin/step-ca
curl -LO https://dl.step.sm/gh-release/cli/docs-ca-install/v0.19.0/step_linux_0.19.0_amd64.tar.gz
tar -xzvf step_linux_0.19.0_amd64.tar.gz
mv step_0.19.0/bin/step /usr/local/bin
chown root:root /usr/local/bin/step
restorecon /usr/local/bin/step
rm -rf step*
```

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
- Create the CSR for the intermediate_ca

```bash
step certificate create <subject-of-cert> intermediate.csr ~/.step/secrets/intermediate_ca_key --csr [--san subca01.example.com [--san subca01.example.com]] --password-file ~/.step/key.pass
```

- Copy the CSR to the root_ca and sign it
  - For ADCS it looks like that for signing

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
- Fix the permissions for both directories

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
- And finally open the firewall

```bash
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

- To use some `step` commands after the move you will need to add some variables to your profile (like the `~/.bashrc`)

```bash
export STEPPATH=/etc/step-ca
export STEP_CA_URL=https://subca01.example.com
export STEP_ROOT=/etc/step-ca/certs/root_ca.crt
```

## Active revocation

[Enable Active Revocation](https://smallstep.com/docs/step-ca/certificate-authority-server-production#enable-active-revocation-on-your-intermediate-ca)

If active revocation is needed the step for the sub-ca creation works a little bit different.

- Before creating the CSR create a template file in `~/.step/templates/intermediate.tpl` with the following content

```json
{
        "subject": {{ toJson .Subject }},
        "keyUsage": ["certSign", "crlSign"],
        "basicConstraints": {
                "isCA": true,
                "maxPathLen": 0
        },
        "crlDistributionPoints": {
                ["http://crl.example.com/crl/subca01.crl"]
        }
}
```

- After that run the this command instead of the other CSR create

```bash
step certificate create <subject-of-cert> intermediate.csr ~/.step/secrets/intermediate_ca_key --csr [--san <subject-of-cert>.example.com [--san subca01.example.com]] --password-file ~/.step/key.pass --template intermediate.tpl
```

- Create the CRL manually i.e. with openssl
  - Create a openssl config file in `~/openssl.conf`

    ```ini
    [ ca ]
    default_ca = CA_default

    [ CA_default ]
    default_crl_days = 30
    database = index.txt
    default_md = sha256
    ```

  - Create an empty `index.txt` file with `touch index.txt`
  - Create the CRL in PEM form

    ```bash
    openssl ca -config openssl.conf -gencrl -keyfile ~/.step/secrets/intermediate_ca_key -cert ~/.step/certs/intermediate_ca.crt -out ca.crl.pem
    ```

  - Convert the CRL to DER form

    ```bash
    openssl crl -inform PEM -in ca.crl.pem -outform DER -out ca.crl
    ```

- And finally upload the CRL file to the webserver which you defined in the `crlDistributionPoints` property
- If a certificate needs to be revoked manually enter it in the index.txt file and renew the CRL

The CRL will expire, so don't forget to renew it manually or automatically from time to time!
