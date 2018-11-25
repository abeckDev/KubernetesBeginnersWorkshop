# KubernetesBeginnersWorkshop

**Still in Development**
*This is just an early draft to not loose some notes*

A beginners workshop for Kubernetes which contains a simple demo setup of the cluster itself and the provisioning resources with Kubernetes files and Helm.


## Prerequisites


## Lab Configuration

In this Lab we will use 4 Virtual Box machines. The VMs are all based on CentOs 7.5 and they are all connected to the Virtual Box NAT Adapter and an internal Virtual Box Network. We will use one as the Kubernetes Master and the other VMs as the Worker for the Cluster.

### Network settings

#### Kubernetes Master

The Kubernetes master is the brain in our cluster and coordinates the worker nodes. Normally you should at least operate three of them to ensure a High available Cluster.

|Setting        |Value                          |
|---------------|-------------------------------|
|IpAddress      | 10.2.10.10                    |
|Netmask        | 255.255.255.0                 |
|Hostname       | kubernetes-master.localdomain |

#### Kubernetes Worker1


|Setting        |Value                          |
|---------------|-------------------------------|
|IpAddress      | 10.2.10.21                    |
|Netmask        | 255.255.255.0                 |
|Hostname       | worker01.localdomain          |

#### Kubernetes Worker2

|Setting        |Value                          |
|---------------|-------------------------------|
|IpAddress      | 10.2.10.22                    |
|Netmask        | 255.255.255.0                 |
|Hostname       | worker02.localdomain          |

#### Kubernetes Worker3

|Setting        |Value                          |
|---------------|-------------------------------|
|IpAddress      | 10.2.10.23                    |
|Netmask        | 255.255.255.0                 |
|Hostname       | worker03.localdomain          |


### Firewall settings

The following settings need to be configured in the firewall.

To open the firewall for a service you can use the following line:

```bash
firewall-cmd --add-service <serviceName> --permanent
```

To open the firewall for a port you can use the following line:

```bash
firewall-cmd --add-port <portNumber>/tcp|udp --permanent
```

Dont forget to load the new ruleset after setting the corresponding rules with the following command:

```bash
firewall-cmd --reload
```

## Initial Setup (all machines)

### Install updates

```bash
yum update -y
```

### Install needed tools

```bash
yum install vim net-tools ntp
```

### Disable SELinux

```bash
vim /etc/selinux/config

SELINUX=disabled
```

### Name resolution

Add the following lines to /etc/hosts on each machine

```bash
10.2.10.10  kubernetes-master kubernetes-master.localdomain
10.2.10.21  worker01 worker01.localdomain
10.2.10.22  worker02 worker02.localdomain
10.2.10.23  worker03 worker03.localdomain
```

### Time syncronisation

Add the following lines to your /etc/ntp.conf

```bash
# NTP Timeserver
server 0.pool.ntp.org minpoll 4 maxpoll 4 burst iburst
server 1.pool.ntp.org minpoll 4 maxpoll 4 burst iburst
server 2.pool.ntp.org minpoll 4 maxpoll 4 burst iburst
server 3.pool.ntp.org minpoll 4 maxpoll 4 burst iburst



# dont panic if large time span
tinker panic 0


# Set the cluster member as peers
peer kubernetes-master minpoll 4 maxpoll 4 burst iburst
peer worker01 minpoll 4 maxpoll 4 burst iburst
peer worker02 minpoll 4 maxpoll 4 burst iburst
peer worker02 minpoll 4 maxpoll 4 burst iburst
```

Start the ntpd daemon

```bash
ntpdate 0.de.pool.ntp.org
systemctl start ntpd
systemctl enable ntpd
```

### Disable swap

Kubeadm doesnt like swap 

```bash
# Get swap with
cat /proc/swaps

#Comment swap line in /etc/fstab out
vim /etc/fstab

#/dev/mapper/centos_kubernetes--master-swap swap                    swap    defaults        0 0
```

### Add Kubernetes Repo

Add the following to the new /etc/yum.repos.d/kubernetes.repo file.

```bash
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
```

After that make yum cache.

```bash
yum makecache
```

### Reboot

```bash
shutdown -r now "System is going down for update"
```

After reboot check

```bash
getenforce # should be disabled
date # schould be exact the same on every machine
```

## Kubernetes Master Setup

### Firewall Setup

```bash
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp

firewall-cmd --add-port 6784/tcp --permanent
firewall-cmd --add-port 6784/udp --permanent
firewall-cmd --add-port 6783/udp --permanent
firewall-cmd --add-port 6783/tcp --permanent
firewall-cmd --reload
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### Install Kubeadm and Docker

```bash
yum install kubeadm docker -y
```

### Start dependencies for Kubernetes setup 

```bash
systemctl restart docker && systemctl enable docker
systemctl restart kubelet && systemctl enable kubelet
```

### Initialize master with Kubeadm

```bash
kubeadm init --apiserver-advertise-address "10.2.10.10" # Remember the token given to you by the installer
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Deploy a pod network to the cluster (WeaveNet)

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### Validate K8s master

Every pod shoud be up and doing great.

```bash
kubectl get pods --all-namespaces
```

## Setup Worker (on all)

### Firewall settings 

```bash
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd --permanent --add-port=6783/tcp
firewall-cmd --add-port 6784/tcp --permanent
firewall-cmd --add-port 6784/udp --permanent
firewall-cmd --add-port 6783/udp --permanent
firewall-cmd --add-port 6783/tcp --permanent
firewall-cmd  --reload
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

### Install Kubeadm and start dependencies

```bash
yum  install kubeadm docker -y
systemctl restart docker && systemctl enable docker
```

### Join Kubeadm Cluster

```bash
kubeadm join 10.2.10.10:6443 --token <TokenFromInstaller> --discovery-token-ca-cert-hash <DiscoveryToken>
```

### Check Status on Master

```bash
kubectl get nodes
```