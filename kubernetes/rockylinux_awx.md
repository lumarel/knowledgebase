# Ansible AWX on a Single node Kubernetes cluster on Rocky Linux 8

## Deploy VM

- Deploy Rocky Linux 8 ISO with 4 cores, 12Gi of memory, 50Gi of storage
- Configure A record for machine and CNAME record for AWX on DNS server
- On install configure fqdn as hostname and given static IP address
- Also configure the NTP server in Anaconda
- Install system helper tools `dnf install dnf-utils setroubleshoot-server`
- Do all updates with `dnf update`
- Disable swap with `swapoff -a` and remove the configuration from the fstab
- Disable the firewall, don't have a solution for enabled firewall up to now
- Reboot

## Install Kubernetes

- Enable kernel modules:

```bash
modprobe br_netfilter
modprobe overlay
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack
cat <<EOF | tee /etc/modules-load.d/k8s_kernel_modules.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack

EOF
```

- Configure sysctl:

```bash
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.ipv4.ip_forward=1
cat <<EOF | tee /etc/sysctl.d/01-k8s.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1

EOF
```

- Add the docker repo `dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
- Install docker `dnf install docker-ce`
- Configure the docker application with `mkdir -p /etc/docker` and add the following content:

```bash
cat <<EOF | tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "insecure-registries": [
     "k8s.gcr.io",
     "docker.io",
     "quay.io",
     "registry.access.redhat.com"
  ]
}

EOF
```

- Enable the docker service `systemctl enable --now docker`
- Add the kubernetes repo:

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

EOF
```

- Install Kubernetes `dnf install kubeadm kubectl`
- Create the kubeconfig with:

```bash
cat <<EOF | tee ~/kubeconfig.yml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration

---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v<current-kubeadm-version>
networking:
  podSubnet: 10.16.0.0/16
  serviceSubnet: 10.96.0.0/12

EOF
```

- Start the kubelet `systemctl enable --now kubelet.service`
- Pre-download the images `kubeadm config images pull --config kubeconfig.yml`
- Init the kubernetes cluster `kubeadm init --ignore-preflight-errors SystemVerification --skip-token-print --config kubeconfig.yml`
- Copy the running config with:

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

- Remove the node taints for the single-node-cluster `kubectl taint nodes --all node-role.kubernetes.io/master-`
- Make sure all pods except the 2 coredns ones are running
- Do `systemctl edit kubelet.service` and write the following into the spacing section

```ini
[Service]
CPUAccounting=true
MemoryAccounting=true
```

- Restart the kubelet `systemctl restart kubelet.service`
- Download the sources of all needed system services:

```bash
curl -LO https://github.com/vmware-tanzu/antrea/releases/download/v<latest-version>/antrea.yml
curl -LO https://raw.githubusercontent.com/rancher/local-path-provisioner/v<latest-version>/deploy/local-path-storage.yaml
curl -LO https://projectcontour.io/quickstart/contour.yaml
```

- Install antrea `kubectl apply -f antrea.yml`
- Make sure that all antrea and coredns pods are running `watch kubectl get po -n kube-system`
- Create the local-path-storage folder `mkdir -p /opt/local-path-provisioner`
- Install the local-path-storage provider `kubectl apply -f local-path-storage.yaml`
- Make sure the pod is running and the storageclass got created `watch kubectl get sc`
- Make this provider to the default one `kubectl patch sc local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
- Install contour `kubectl apply -f contour.yaml`
- Make sure that all contour pods are running `watch kubectl get po -n projectcontour`

## Install AWX

- Install missing packages `dnf install git make`
- Clone the awx-operator `git clone https://github.com/ansible/awx-operator.git`
- Checkout the latest release `cd awx-operator` and `git checkout tags/<latest-version>`
- Set the namespace in which the awx-operator and awx will be deployed `export NAMESPACE=default`
- Deploy the awx-operator `make deploy`
- Make sure the operator is running `watch kubectl get po`
- Either create a new self-signed certificate (`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out ingress-tls.crt -keyout ingress-tls.key -subj "/CN=<awx-fqdn>/O=awx-ingress-tls"`) or copy a real one to the machine (ingress-tls.crt and ingress-tls.key need to be only the server certificate without an empty line)
- Import the certificate into kubernetes `kubectl create secret tls awx-ingress-tls --key ingress-tls.key --cert ingress-tls.crt`
- Download the root certificate and import it:

```bash
curl -O http://certificates.fritz.box/rootca_public.crt
kubectl create secret generic awx-custom-certs --from-file=bundle-ca.crt=/root/rootca_public.crt
```

- Create the AWX config file for Kubernetes:

```bash
cat <<EOF | tee ~/awx.yml
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
  hostname: <awx-fqdn>
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

EOF
```

- Deploy the AWX with `kubectl apply -f ~/awx.yml`
- Make sure the pods got deployed with `kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager`
- And `watch kubectl get ing,po,svc,pvc`

## Upgrade components

### Ansible AWX Upgrade

- `cd awx-operator`
- `git reset --hard devel`
- `git checkout tags/<latest-version>`
- `make deploy`
- `kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager`
- `watch kubectl get ing,po,svc,pvc`

### Antrea Upgrade

[https://github.com/antrea-io/antrea/blob/main/docs/versioning.md#antrea-upgrade-and-supported-version-skew](https://github.com/antrea-io/antrea/blob/main/docs/versioning.md#antrea-upgrade-and-supported-version-skew)

Upgrade at max 4 minor versions

- `mv antrea.yml antrea.yml.1`
- `curl -LO https://github.com/vmware-tanzu/antrea/releases/download/v<latest-version>/antrea.yml`
- `kubectl apply -f antrea.yml`
- `watch kubectl get po -n kube-system`

### Contour Upgrade

[https://projectcontour.io/resources/upgrading/](https://projectcontour.io/resources/upgrading/)

- `mv contour.yaml contour.yaml.1`
- `curl -LO https://projectcontour.io/quickstart/contour.yaml`
- `kubectl delete namespace projectcontour`
- `kubectl apply -f contour.yaml`
- `watch kubectl get po -n projectcontour`

## Troubleshooting

### Antrea Problems

```bash
kubectl logs -n kube-system antrea-agent-<key> -c antrea-agent
```

### Contour Problems

```bash
kubectl logs -n projectcontour deployment/contour --all-containers -f
```
