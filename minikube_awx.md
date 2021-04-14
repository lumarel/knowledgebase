# Ansible AWX installation on Minikube

## System setup

- Install CentOS 8 machine with local wheel user
- Add docker-ce repo
- Install docker and conntrack

  `dnf install docker-ce conntrack`

- Enable and start docker

  `systemctl enable --now docker`

- Add local user to docker group

  `sudo usermod -a -G wheel,docker <local user>`

- Download and install minikube rpm

  `curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm`

  `sudo rpm -ivh minikube-latest.x86_64.rpm`

- Configure firewall

  `sudo firewall-cmd --permanent --add-masquerade --zone=public`

  `sudo firewall-cmd --reload`

- Set kubernetes alias (and add this line to `~/.bashrc`)

  `alias k="minikube kubectl --"`

- Disable SELINUX for installation until next boot

  `sudo setenforce 0`

- Startup minikube

  `minikube start --addons=ingress --driver=none`

- Install awx-operator

  `k apply -f https://raw.githubusercontent.com/ansible/awx-operator/devel/deploy/awx-operator.yaml`

- Wait until awx-operator is up

  `k logs -f deployment/awx-operator`

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

  `k get validatingwebhookconfigurations`

  `k delete -A ValidatingWebhookConfiguration ingress-nginx-admission`

- Install awx

  `k apply -f my-awx-password.yml`

  `k apply -f my-awx.yml`

- Check that everything works
- Stop minikube

  `minikube stop`

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

  `sudo systemctl daemon-reload`

  `sudo systemctl start minikube.service`
