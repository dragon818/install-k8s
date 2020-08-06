# install-k8s


# Use kubeadm Install Centos k8s

base on Centos7   kernel 4.4.232

## 1.Set hostname

```
[root@xxx ~]# hostnamectl set-hostname XXX
```

```hostname list 
192.168.189.10 k8s-master01
192.168.189.20 k8s-node01
192.168.189.30 k8s-node02
```

## 2.Set Ip 

```powershell
[root@xxx ~]# cat /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
UUID="84915e90-9252-45a5-b705-4ca5d133978e"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.189.10" 
NETMASK="255.255.255.0"
GATEWAY="192.168.189.2"
DNS1="192.168.189.2"
DNS2="8.8.8.8"
```

```powershell
 [root@xxx ~]# systemctl restart network
```

## 3.Set hosts

```powershell
[root@xxx ~]# vi /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

## 4. Turnoff firewall and selinux

### 4.1 Turnoff firewall

```powershell
[root@xxx ~]# systemctl stop firewalld
[root@xxx ~]# systemctl disable firewalld

check it 

[root@xxx ~]# firewall-cmd --state

not running

```

### 4.2 Turnoff selinux



```powershell
check status

[root@xxx ~]# getenforce 
Enforcing
```

```powershell
[root@xxx ~]# setenforce 0
```

```powershell
[root@xxx ~]# sed -i 's/SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

```powershell
[root@xxx ~]# reboot
```



```powershell
check status again

[root@xxx ~]# getenforce 
Permissvie
```

## 5. Sync time

```powershell
[root@xxx ~]# yum -y install ntpdate
```

```powershell
[root@xxx ~]# crontab -e

0 */1 * * * ntpdate time1.aliyun.com
```

```powershell
[root@xxx ~]# ntpdate time1.aliyun.com
```



## 6. Turnoff swap forever

```powershell
[root@xxx ~]# vi /etc/fstab

#
# /etc/fstab
# Created by anaconda on Thu Jul 30 10:48:43 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=e7c465d4-cd5d-4492-a28a-6299147706d7 /boot                   xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
# /dev/mapper/centos-swap swap                    swap    defaults        0 0

```

```
# is annotation, add # at the beginning of the line
```

```powershell
[root@xxx ~]# reboot
[root@xxx ~]# free -m
```

## 7.  Set Netfilter

```powershell
[root@xxx ~]# vi /etc/sysctl.d/kubernets.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1

[root@xxx ~]# sysctl -p /etc/sysctl.d/kubernets.conf

check =======================
```

```
[root@xxx ~]# modprobe br_netfilter 
```

## 8.Turnoff postfix

```
systemctl stop postfix && systemctl disable postfix
```

## 8.Set journald

```
mkdir /var/log/journal
mkdir /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]

Storage=persistent

Compress=yes

SyncIntervalSec=5m

RateLimitInterval=30s

RateLimitBurst=1000

SystemMaxUse=10G

SysteMaxFileSize=20M

MaxRetentionSec=2week

ForwardToSyslog=no
EOF

systemctl restart systemd-journald
```



## 8.Install ipvs

```powershell
[root@xxx ~]# yum -y install ipset ipvsadm

[root@xxx ~]# cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
```

```powershell
[root@xxx ~]# chmod 755 /etc/sysconfig/modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

## 9. Install Docker

```powershell
# (Install Docker CE)
## Set up the repository
### Install required packages
[root@xxx ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
```

```powershell
## Add the Docker repository
[root@xxx ~]# yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

```
# Install Docker CE
yum update -y && yum install -y docker-ce
  
```

```
## Create /etc/docker
mkdir /etc/docker
```

```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```



```
mkdir -p /etc/systemd/system/docker.service.d
```



```
# Restart Docker
systemctl daemon-reload
systemctl restart docker
```

```
systemctl enable docker
```



## 10.Install k8s

```powershell
[root@xxx ~]# vim /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

```powershell
[root@xxx ~]# yum install -y kubelet kubeadm kubectl


[root@xxx ~]# kubeadm config print init-defaults

apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.18.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

```
check /var/lib/kubelet/config.yaml file    --systemd-- cgroupdriver
```

```powershell
[root@xxx ~]# systemctl enable kubelet
```

## 11. Pull Images for k8s

### 11.1 just for master node 

```
kubeadm config images pull
```

```

```



### 11.2  Copy images from master to worknode 

pause and kube-proxy

```
docker save -o filename.tar   name:version
```

```
docker load -i tarfile
```



## 12. Init k8s 

```
kubeadm init --kubernetes-version=v1.18.6 --pod-network-cidr=172.16.0.0/16 --apiserver-advertise-address=192.168.189.10
```



Result of the initial file

```
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.189.10]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master01 localhost] and IPs [192.168.189.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master01 localhost] and IPs [192.168.189.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0805 15:06:30.620317   10788 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0805 15:06:30.622157   10788 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.007616 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: o3g0u1.hwlg08bg8tlq90sg
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.189.10:6443 --token o3g0u1.hwlg08bg8tlq90sg \
    --discovery-token-ca-cert-hash sha256:8022504cdab0a6e9146be42ba661eab1912c17637da6c0ed9d02c727a99bec15 
[root@k8s-master01 ~]# 
```



## 13.Install flannel for master



```
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

kubectl apply -f kube-flannel.yml
```

* reset the Network in the yml file  to "172.16.0.0/16" this is the pod-network





```
kubeadm join 192.168.189.10:6443 --token o3g0u1.hwlg08bg8tlq90sg \
    --discovery-token-ca-cert-hash sha256:8022504cdab0a6e9146be42ba661eab1912c17637da6c0ed9d02c727a99bec15 
```

```
kubectl get node
```





```
vim 01-create-deployment-nigix-app1-service.yaml
```

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-apps
spec:
  replicas: 2
  selector:
    matchLabels:
      apps: nginx
  template:
    metadata:
      labels:
        apps: nginx
    spec:
      containers:
      - name: nginxapp1
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app1-svc
spec:
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    apps: nginx
```



```
kubectl apply -f 01-create-deployment-nigix-app1-service.yaml
```



```
kubectl get deployment
kubectl get svc 
kubectl get pod -o wide
kubectl get endpoints

kubectl exec -it podname bash
exit
```

```
kubectl describe pod podname

kubectl logs podName -c containerName
```









