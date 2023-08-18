# Ansible AWX on a Single node Kubernetes cluster on PhotonOS 4

[!WARNING]
Deprecated guide

[https://virtual-bytes.co.uk/2021/05/06/deploying-ansible-awx-on-kubernetes-from-day-0](https://virtual-bytes.co.uk/2021/05/06/deploying-ansible-awx-on-kubernetes-from-day-0/)

## *This guide is outdated, since at least Kubernetes 1.24*

## Deploy VM

- Deploy PhotonOS 4.0 ISO with 4 cores, 8Gi of memory, 50Gi of storage
- Configure A record and CNAME record on DNS server
- On install configure fqdn as hostname and given static IP address
- Configure `/etc/systemd/timesyncd.conf` `NTP=` in the `[Time]` section
- Restart `systemd-timesyncd.service` after configuring
- Do all updates with `tdnf update`
- Reboot

## Install Kubernetes

- `tdnf install kubernetes-kubeadm`
- `systemctl disable --now iptables`
- `systemctl enable --now docker`
- Write the following into `kubeconfig.yml`

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration

---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.19.0
networking:
  podSubnet: 10.16.0.0/16
  serviceSubnet: 10.96.0.0/12

```

- `kubeadm config images pull --config kubeconfig.yml`
- `kubeadm init --ignore-preflight-errors SystemVerification --skip-token-print --config kubeconfig.yml`
- `mkdir -p $HOME/.kube`
- `cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`
- `chown $(id -u):$(id -g) $HOME/.kube/config`
- `kubectl taint nodes --all node-role.kubernetes.io/master-`
- `watch kubectl get po -A`
- Do `systemctl edit kubelet.service` and write the following into the spacing section

```ini
[Service]
CPUAccounting=true
MemoryAccounting=true
```

- `systemctl restart kubelet.service`
- `curl -LO https://github.com/vmware-tanzu/antrea/releases/download/v1.2.3/antrea.yml`
- `kubectl apply -f antrea.yml`
- `watch kubectl get po -n kube-system`
- `mkdir -p /opt/local-path-provisioner`
- `curl -LO https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.20/deploy/local-path-storage.yaml`
- `kubectl apply -f local-path-storage.yaml`
- `watch kubectl get sc`
- `kubectl patch sc local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
- `curl -LO https://projectcontour.io/quickstart/contour.yaml`
- `kubectl apply -f contour.yaml`
- `watch kubectl get po -n projectcontour`

## Install awx

- `tdnf install git make tar`
- `git clone https://github.com/ansible/awx-operator.git`
- `cd awx-operator`
- `git checkout tags/0.14.0`
- `export NAMESPACE=default`
- `make deploy`
- `watch kubectl get po`
- `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out ingress-tls.crt -keyout ingress-tls.key -subj "/CN=awx.fritz.box/O=awx-ingress-tls"`
- `kubectl create secret tls awx-ingress-tls --key ingress-tls.key --cert ingress-tls.crt`
- `curl -O http://certificates.fritz.box/rootca_public.crt`
- `kubectl create secret generic awx-custom-certs --from-file=bundle-ca.crt=/root/rootca_public.crt`
- Write the following into `awx.yml`

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-admin-password
  namespace: default
stringData:
  password: <password>

---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ingress_type: Ingress
  hostname: awx.fritz.box
  ingress_tls_secret: awx-ingress-tls
  web_resource_requirements:
    requests:
      cpu: 400m
      memory: 2Gi
    limits:
      cpu: 1000m
      memory: 4Gi
  task_resource_requirements:
    requests:
      cpu: 250m
      memory: 1Gi
    limits:
      cpu: 500m
      memory: 2Gi
  bundle_cacert_secret: awx-custom-certs

```

- `kubectl apply -f awx.yml`
- `kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager`
- `watch kubectl get ing,po,svc,pvc`

## Upgrade components

### Ansible AWX

- `cd awx-operator`
- `git reset --hard devel`
- `git fetch --all`
- `git checkout tags/0.14.0`
- `make deploy`
- `kubectl logs -f deployments/awx-operator-controller-manager -c manager`
- `watch kubectl get ing,po,svc,pvc`

### Antrea

[https://github.com/antrea-io/antrea/blob/main/docs/versioning.md#antrea-upgrade-and-supported-version-skew](https://github.com/antrea-io/antrea/blob/main/docs/versioning.md#antrea-upgrade-and-supported-version-skew)

Upgrade at max 4 minor versions

- `mv antrea.yml antrea.yml.1`
- `curl -LO https://github.com/vmware-tanzu/antrea/releases/download/v1.2.3/antrea.yml`
- `kubectl apply -f antrea.yml`
- `watch kubectl get po -n kube-system`

### Contour

[https://projectcontour.io/resources/upgrading/](https://projectcontour.io/resources/upgrading/)

- `mv contour.yaml contour.yaml.1`
- `curl -LO https://projectcontour.io/quickstart/contour.yaml`
- `kubectl delete namespace projectcontour`
- `kubectl apply -f contour.yaml`
- `watch kubectl get po -n projectcontour`
