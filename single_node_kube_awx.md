# Ansible AWX on a Single node Kubernetes cluster on PhotonOS 4

[https://virtual-bytes.co.uk/2021/05/06/deploying-ansible-awx-on-kubernetes-from-day-0](https://virtual-bytes.co.uk/2021/05/06/deploying-ansible-awx-on-kubernetes-from-day-0/)

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
- `curl -LO https://github.com/vmware-tanzu/antrea/releases/download/v1.1.2/antrea.yml`
- `kubectl apply -f antrea.yml`
- `watch kubectl get po -A`
- `mkdir -p /opt/local-path-provisioner`
- `curl -LO https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.20/deploy/local-path-storage.yaml`
- `kubectl apply -f local-path-storage.yaml`
- `watch kubectl get sc`
- `kubectl patch sc local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
- `curl -LO https://projectcontour.io/quickstart/contour.yaml`
- `kubectl apply -f contour.yaml`
- `watch kubectl get po -n projectcontour`

## Install awx

- `curl -LO https://raw.githubusercontent.com/ansible/awx-operator/0.13.0/deploy/awx-operator.yaml`
- `kubectl apply -f awx-operator.yaml`
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
- `kubectl logs -f deployments/awx-operator`
- `watch kubectl get ing,po,svc,pvc`
