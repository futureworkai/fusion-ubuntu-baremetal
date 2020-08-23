
# Fusion 5.x on Single Node Bare Metal Kubernetes

This repo contains scripts for installing Fusion 5.x on Kubernetes (K8s).
The scripts provide an option to create Kubernetes clusters that are suitable for demo / proof-of-concept purposes only.
We assume that you'll want to control how your production clusters are provisioned, secured, and managed, as these are typically concerns we're not able to script for you.

---

## Prerequisites

32 Gb Plus of RAM Not Tested with Less
Internet Connectivity Assumed no proxy required
100 Gb of Free Disk Space not tested with less (20 Gb will be enough for empty installation)

---

Author <hello@futurework.ai>

[] Step 1. Bare Metal Ubuntu 18.04 LTS Install
[] Step 2. Prep an Ubuntu 18.04 LTS Bare Metal for Single Node Kubernes
[] Step 3. Install Single Node Kubernetes
[] Step 4. Install Fusion 5.1.2

> NOTE: This is a Fork of the Complete fusion-cloud-native Repo
<https://github.com/lucidworks/fusion-cloud-native>

This Script and Instruction Set is only intended for and Tested on a Bare Metal
Installation of Ubuntu Server 18.04.5

---

## Step 1. Bare Metal Ubuntu

This can be tweaked for your Specific Hardware, noting that
the Disk Congiguration/Auto Formatting is the **DANGEROUS**
bit you really need to sort out yourself.

For purposes of this installation guide call your host

fuse

Create your User account as

Root Enabled User:
mrfusion

Password:
backintime

For IP Address it's best you have a DHCP reservation setup
for this MAC Address or configure your own settings statically

And only install the SSH package during the GUI

---

## Step 2. Prepare Ubuntu for K8S

Open an SSH session to your Box I like MobaTerm but Putty etc will also work

nano ~/step2-prepare-ubuntu-for-k8s.sh

> Cut and Paste the Contents into your file, Ctrl-X -> Save
> chmod +x ./step2-prepare-ubuntu-for-k8s.sh

sudo ./step2-prepare-ubuntu-for-k8s.sh

```bash
# Remove or comment entry for /swap.img
#vi /etc/fstab
sudo sed -i 's/\/swap.img/\#\/swap.img/g' /etc/fstab
# Disable Swap
sudo swapoff -av
# Delete the Swap File
sudo rm /swap.img
# Do Updates <- No Prompt etc...
sudo apt-get update
sudo apt-get --assume-yes upgrade
sudo apt-get --assume-yes install git
sudo apt-get --assume-yes install nfs-kernel-server
sudo mkdir -p /srv/k8s
sudo chown nobody:nogroup /srv/k8s
# Everyone/Anyone Can Access
sudo chmod 777 /srv/k8s
# Edit /etc/exports and add the following line:
# https://www.thegeekdiary.com/understanding-the-etc-exports-file/
sudo cat > /etc/exports <<EOF
/srv/k8s 127.0.0.1(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
EOF
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

---

## Step 3. Install Native Full K8S as Single Node

nano ~/step3-install-configure-k8s.sh

> Cut and Paste the Contents into your file, Ctrl-X -> Save
> chmod +x ./step3-install-configure-k8s.sh

sudo ./step3-install-configure-k8s.sh

```bash
sudo apt-get update && apt-get install -y \
sudo apt-transport-https ca-certificates curl software-properties-common
gnupg2
# Add Docker’s official GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
# Add Docker apt repository.
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"
# Install Docker CE
sudo apt-get update && apt-get install -y \
containerd.io=1.2.13-1 \
docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)
# Setup daemon.
sudo cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2"
}
EOF
sudo mkdir -p /etc/systemd/system/docker.service.d
# Restart docker.
sudo systemctl daemon-reload
sudo systemctl restart docker
# Install kubeadm, kubelet and kubect
# NOTE: We use this exact version not the current as it's known good with Fusion 5.1.2
sudo apt-get update && apt-get install -y apt-transport-https curl
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
sudo cat <<EOF | tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet=1.16.9-00 kubeadm=1.16.9-00 kubectl=1.16.9-00
sudo apt-mark hold kubelet kubeadm kubectl
# Create a cluster
# Using 10.8.8.x for K8s internal IP addresses (pods).
# Get Docker Content and Initialise Control Plane
# Verify connectivity and pull from gcr.io container image registry
sudo kubeadm config images pull
# Initialize the control-plane node
# If you intend on creating additional nodes and joining them,
# you'll want to save the kubeadm join command which gets echoed
sudo kubeadm init --pod-network-cidr=10.8.8.0/24
### These commands allow your non-root-user to interact with k8s
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# Rename context to remove "@" symbol since it causes issues
# This is the name of the cluster we'll use with setup_f5_k8s.sh later
sudo kubectl config rename-context kubernetes-admin@kubernetes cold-k8s
# Install Calico
wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml
# Change value for CALICO_IPV4POOL_CIDR to 10.8.8.0/24
# 192.168.0.0/16 <- this was the original value it appears once so a straight replace is safe
# Edit via sed The 1980s are still the best ways
sed -i 's/192.168.0.0\/16/10.8.8.0\/24/g' calico.yaml
# Apply the Template
sudo kubectl apply -f calico.yaml
# Remove taint for isolation of control-plane node
# By default, the master control-plane node is not schedule-able for general purpose pods (such as for Fusion). 
# Removing this taint allows us to run everything on a single node.
# NOTE: This is why this is not PRODUCTION suitable only for DEV/TEST/DEMO Usage
sudo kubectl taint nodes --all node-role.kubernetes.io/master-
# Install helm v3
sudo snap install helm --classic
# Add stable repository for helm v3
helm repo add stable https://kubernetes-charts.storage.googleapis.com
# Install NFS Client Provisioner into kube-system namespace
helm install nfs-client-provisioner stable/nfs-client-provisioner \
-n kube-system --set nfs.server=127.0.0.1 --set nfs.path=/srv/k8s \
--set storageClass.defaultClass=true --set storageClass.archiveOnDelete=false
# NOTE Optional for Metrics Uncomment Next Line to Add
#kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
# Now We Check Status with this Command
# TODO: Add a Parser for what it should look like
# NOTE Grep for Init:Error and Init:CrashLoopBackOff
# NOTE For Now We just going to assume it has worked.
sudo kubectl get pods --all-namespaces
```

---

## Step 4. Installing Fusion 5.x Platform

git clone https://github.com/futureworkai/fusion-ubuntu-baremetal
cd fusion-ubuntu-baremetal

# NOTE HERE BE THE MAGIC

```bash
sudo ./setup_f5_k8s_singlenode.sh -c cold-k8s -r cold-fusion -n cold-fusion
```

> Setup Ingress

If you run kubectl get svc/proxy, you’ll notice that you have no external IP address for your Fusion instance:

```bash
sudo kubectl get svc/proxy
```
