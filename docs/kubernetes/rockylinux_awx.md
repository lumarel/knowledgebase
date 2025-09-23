# Ansible AWX on a Single node Kubernetes cluster on Rocky Linux

This guide should work with all versions of Rocky Linux.  
Run all commands as root.

## Deploy VM

- Deploy Rocky Linux ISO with 4 cores, 12Gi of memory, 50Gi of storage
- Configure an A record for the machine and a CNAME record for AWX on your DNS server
- On install configure the fqdn as hostname and given static IP address
- Also configure the NTP server in Anaconda
- (optional) Install system helper tools `dnf install dnf-utils setroubleshoot-server` for ongoing troubleshooting
- Do all updates with `dnf update`
- Disable swap with `swapoff -a` and remove the configuration from the fstab
- Disable the firewall with `systemctl disable --now firewalld`, don't have a solution for enabled firewall up to now
- Reboot

## Install Kubernetes

- Enable kernel modules:

```bash
modprobe br_netfilter
modprobe overlay
cat <<EOF | tee /etc/modules-load.d/k8s_kernel_modules.conf
overlay
br_netfilter

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

- Add the cri-o repo:

```bash
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v<stable-version>/rpm/
enabled=1
gpgcheck=1
gpgkey=https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v<stable-version>/rpm/repodata/repomd.xml.key

EOF
```

- Add the kubernetes repo (make sure you use the same version as for cri-o):

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v<stable-version>/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v<stable-version>/rpm/repodata/repomd.xml.key

EOF
```

- Install cri-o and Kubernetes `dnf install container-selinux cri-o cri-tools kubeadm kubectl kubelet`
- Enable the cri-o service `systemctl enable --now crio`
- Create the kubeconfig with:

```bash
cat <<EOF | tee ~/kubeconfig.yml
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v<installed-kubeadm-version>
networking:
  podSubnet: 10.16.0.0/16
  serviceSubnet: 10.96.0.0/12

EOF
```

- Start the kubelet `systemctl enable --now kubelet.service`
- Pre-download the images `kubeadm config images pull --config kubeconfig.yml`
- Init the kubernetes cluster `kubeadm init --skip-token-print --config kubeconfig.yml`
- Copy the running config with:

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

- Remove the node taints for the single-node-cluster:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

- Make sure all pods *except* the 2 coredns ones are running
- Do `systemctl edit kubelet.service` and write the following into the spacing section

```ini
[Service]
CPUAccounting=true
MemoryAccounting=true
```

- Restart the kubelet `systemctl restart kubelet.service`
- Download the sources of all needed system services:

```bash
curl -LO https://github.com/antrea-io/antrea/releases/download/v<latest-version>/antrea.yml
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

## Get certificates

### cert-manager / ACME

To be able to use cert-manager for signing certificates by a ACME based CA like step-ca you need to configure a cluster wide issuer.  
The example configuration uses a step-ca.

- Download the source to install cert-manager:

```bash
curl -LO https://github.com/cert-manager/cert-manager/releases/download/v<latest-version>/cert-manager.yaml
```

- Install cert-manager `kubectl apply -f cert-manager.yaml`
- Make sure that all pods are running `watch kubectl get po -n cert-manager`
- Gather your certificate chain (especially the rootca's), and convert them to a base64 string, make sure there are no newlines at the end, this will be used for the caBundle variable
- Create the ClusterIssuer config file:

```bash
cat <<EOF | tee ~/cert-issuer.yml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <ca-name>
  namespace: cert-manager
spec:
  acme:
    email: <email-address>
    privateKeySecretRef:
      name: <ca-name>
    server: https://<ca-fqdn>/acme/<provisioner-name>/directory
    caBundle: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQ0KTUlJQk9nSUJBQUpCQUtqMzR3hGaEQ5MHZjTkxZTEluRkVYNlBweTF0UGY5Q256ajRwNFdHZUtMczFQdDhRdQ0KS1VwUktmRkxmUllDOUFJS2piSlRXaXQrQ3F2aldZenZRd0VDQXdFQUFRSkFJSkxpeEJ5MnFwRm9TNERTbW9FbQ0KbzNxR3kwdDZ6MDlBSUp0SCs1T2VSVjFiZStONGNEWUpLZmZHekRhODh2UUVOWmlSbTBHUnE2YStIUEdRTWQyaw0KVFFJaEFLTVN2eklCbm5pN290L09TaWUG1KTFk0U3dUUUFldlh5c0UyUmJGRFlkQWlFQkNVRWFSUW5NbmJwNw0KOW14RFhEZjZBVTBjTi9SUEJqYjlxU0hEY1daSEd6VUNJRzJFczU5ejZ0dyRFkrcHhMUW53Zm90YWR4ZCtVeQ0Kdi9PdzVUMHE1Z0lKQWlFQXlTNFJhSTlZRzhFV3gvMncwVDY3WlVWQXc4ZU9NQjZCSVVnMFhjdSszb2tDSUJPcw0KLzVPaVBnb1RkU3k3YmNGOUlHcFNFOFpnR0t6Z1lRVlplTjk3WUUwMA0KLS0tLS1FTkQgUlNBIFBSSVZBVEUgS0VZLS0tLS0=
    solvers:
      - http01:
          ingress:
            class: contour

EOF
```

- And apply the config `kubectl apply -f cert-issuer.yml`
- You should be able to verify the config with `kubectl get ciss <ca-name>` now

### Manually signed certificate

- Either create a new self-signed certificate (`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out ingress-tls.crt -keyout ingress-tls.key -subj "/CN=<awx-fqdn>/O=awx-ingress-tls"`) or copy a ca signed one to the machine (ingress-tls.crt and ingress-tls.key need to be only the server certificate without an empty line)
- Import the certificate into kubernetes `kubectl create secret tls awx-ingress-tls --key ingress-tls.key --cert ingress-tls.crt`

## Install AWX

- Install missing packages `dnf install git`
- Download the latest kustomize version from the [Github releases page](https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize%2Fv<latest-version>)
- Unpack, make it a executable and move it to `/usr/local/bin`:

```bash
curl -LO https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv<latest-version>/kustomize_v<latest-version>_linux_amd64.tar.gz
tar xzvf kustomize_v<latest-version>_linux_amd64.tar.gz
chmod +x kustomize
chown root: kustomize
mv kustomize /usr/local/bin
```

- Download the issuing certificates and import them:

```bash
curl -O http://<crl-fqdn>/rootca.crt
kubectl create secret generic awx-custom-certs --from-file=bundle-ca.crt=/root/rootca.crt
```

- Create the Kustomize config file:

```bash
cat <<EOF | tee ~/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=<latest-version>
  - awx.yml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: <latest-version>

# Specify a custom namespace in which to install AWX
namespace: default

EOF
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
  ingress_annotations: |
    cert-manager.io/cluster-issuer: <ca-name>
    ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
  ingress_hosts:
    - hostname: <awx-fqdn>
      tls_secret: <secret-name-used-by-cert-manager>
  ingress_controller: contour
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

In case you are using a manually signed certificate, which you already imported like shown above use the following AWX configuration instead:

```yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ingress_type: Ingress
  ingress_hosts:
    - hostname: <awx-fqdn>
      tls_secret: <secret-name-used-for-tls-secret>
  ingress_controller: contour
  web_resource_requirements:
...
```

- Install the awx-operator (run this step twice, make sure to wait inbetween until the awx-operator is ready) `kubectl apply -k .`
- Make sure the operator is running `watch kubectl get po`
- Make sure the pods got deployed with `kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager`
- And `watch kubectl get ing,po,svc,pvc`

## Upgrade components

### Kubernetes Upgrade

!!! warning
    Upgrading a single node kubernetes cluster is always a play with the fire, make sure you always make backups/snapshots before the operation!

Make sure you always lift the cluster version before you lift the `kubelet` version!

#### Kubernetes cluster version updates

This is only possible if the kubeadm version has progressed, it's only possible to upgrade to versions lower than or exactly the kubeadm version.

- Update the kubeadm version `dnf update kubeadm`
- Check for updates `kubeadm upgrade plan`
- The plan will tell you what is possible to be upgraded (this might show incorrect options, that don't work with your kubeadm version)
- Upgrade to the wanted version with `kubeadm upgrade apply v<cluster-version>`

#### OS updates

- Stop the kubelet, which makes sure etcd doesn't corrupt `systemctl stop kubelet`
- Any update operation, like `dnf update`
- Start kubelet again `systemctl start kubelet`
- Check with `journalctl -f` if the containers are coming up normally again and if the CNI configures the network in a working state again
- If a restart is needed, stop kubelet again and restart (sometimes only a restart gets the system working again)

#### Project repo path switch

The kubernetes project moved from it's home at Google to a community owned location:

[https://kubernetes.io/blog/2023/10/10/cri-o-community-package-infrastructure/](https://kubernetes.io/blog/2023/10/10/cri-o-community-package-infrastructure/)

This means 2 things:

- The repos for both kubernetes and cri-o switched to a different location, look for the exact paths in the guide above
- If you are using cri-o you will have to do a tricky switch, aka you will need to uninstall cri-o and then reinstall it again, as there are file/dependency conflicts between the old and new cri-o version (the whole runtime got merged into the cri-o package now):

```bash
systemctl stop kubelet
dnf remove containers-common
dnf module reset container-tools
dnf install cri-o
systemctl enable --now crio
systemctl start kubelet
```

And there was a second repo switch for cri-o too: [github.com/cri-o/packaging](https://github.com/cri-o/packaging/?tab=readme-ov-file#project-layout)

### Ansible AWX Upgrade

- Change the version in the kustomization.yaml file
- Rerun `kustomize build . | kubectl apply -f -`
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

### CRI-O Problems

There might occur the issue of an `ImageInspectError`, CRI-O does not allow unambiguous image names anymore since 1.34.  
If you still need them make sure to set `short_name_mode = "disabled"` in the crio.conf.

### Antrea Problems

```bash
kubectl logs -n kube-system antrea-agent-<key> -c antrea-agent
```

### Contour Problems

```bash
kubectl logs -n projectcontour deployment/contour --all-containers -f
```

### Local Path Provisioner is unable to create pv's or applications using a pvc can't write

Most likely it's because SELinux is misbehaving.
Could be that you have to set SELinux to permissive mode overall... which is sad.

[https://fedoramagazine.org/kubernetes-with-cri-o-on-fedora-linux-39/](https://fedoramagazine.org/kubernetes-with-cri-o-on-fedora-linux-39/)
