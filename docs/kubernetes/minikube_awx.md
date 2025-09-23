# Ansible AWX installation on Minikube

!!! warning
    Deprecated guide

## System setup

- Install CentOS 8 machine with local wheel user
- Add docker-ce repo
- Install docker and conntrack

```bash
dnf install docker-ce conntrack
```

- Enable and start docker

```bash
systemctl enable --now docker
```

- Add local user to docker group

```bash
sudo usermod -a -G wheel,docker <local user>
```

- Download and install minikube rpm

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
```

- Configure firewall

```bash
sudo firewall-cmd --permanent --add-masquerade --zone=public
sudo firewall-cmd --reload
```

- Set kubernetes alias (and add this line to `~/.bashrc`)

```bash
alias k="minikube kubectl --"
```

- Disable SELINUX for installation until next boot

```bash
sudo setenforce 0
```

- Startup minikube

```bash
minikube start --addons=ingress --driver=none
```

- Install awx-operator

```bash
k apply -f https://raw.githubusercontent.com/ansible/awx-operator/devel/deploy/awx-operator.yaml
```

- Wait until awx-operator is up

```bash
k get pods
```

- Create awx configuration (write a file `my-awx.yml`)

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  tower_ingress_type: Ingress
  tower_hostname: <fqdn of the awx system (CNAME of the host)>
```

- Create awx initial admin password (write a file `my-awx-password.yml`)

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-admin-password
  namespace: default
stringData:
  password: <password of the admin user>
```

- Remove validatingwebhookconfigurations

```bash
k get validatingwebhookconfigurations
k delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```

- Install awx

```bash
k apply -f my-awx-password.yml
k apply -f my-awx.yml
```

- Wait until awx-operator has done its job

```bash
k logs -f deployment/awx-operator
```

- Check that everything works
- Stop minikube

```bash
minikube stop
```

- Create systemd service file (write a file `/etc/systemd/system/minikube.service`)

```ini
[Unit]
Description=Runs minikube on startup
After=docker.service containerd.service

[Service]
Type=oneshot
ExecStart=/usr/bin/minikube start
ExecStop=/usr/bin/minikube stop
User=<local user>
Group=<local user>
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

- Reload systemd and start service

```bash
sudo systemctl daemon-reload
sudo systemctl start minikube.service
```
