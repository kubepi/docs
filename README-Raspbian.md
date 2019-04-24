# Installation and setup instructions for a Kubernetes cluster on Raspberry Pi

## Fundamentals

### Networking

- Understanding NAT

Linux in Action covers this well

## Distributions

Several distributions were trialed for use in this project. They include:

- Hypriot - Found it didn't work and is not maintained.
- Unbuntu Core - Minimal distro. Required to install a classic mode to access common features. Ran into multiple issues as a result of this that weren't worth the pain for what I wanted to do.
- Ubuntu-Mate - Worked fine, but difficulties in updating distribution from 16.04 to 18.04
- Unbuntu Server (ARM64 version)

## Raspian Configurations

Setup of the Raspberry Pis for the most part was achieved following this [guide](http://www.cristiandima.com/running-kubernetes-on-a-raspberry-pi-cluster/)

### Enable SSH

```
sudo raspi-config

# Select Interfacing Options.
# Navigate to and select SSH.
# Choose Yes.
# Select Ok.
# Choose Finish.
```

### Configure a hostname for each device in the cluster

To help identify which node you are working on from the terminal, assign each node a unique hostname.

```
sudo raspi-config

# Select Network Options.
# Select Hostname

# Set as kuber-master, kuber-worker-a & kuber-worker-b
```

### Change default password

```
sudo raspi-config

# Select Change User Password
```

### Static IP

There are 2 approaches that can be taken to set a static IP. For this project, since the intention is to allow it to work on multiple networks (home network along with a travel network i.e. different routers), the preference is to configure a static IP through Raspian itself. However, for future reference both methods will be documented.

1) Static IP through Raspbian

```
sudo nano /etc/dhcpcd.conf
```

**kuber-master**

```
interface eth0
static ip_address=192.168.1.150/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

**kuber-worker-a**

```
interface eth0
static ip_address=192.168.1.151/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

**kuber-worker-b**

```
interface eth0
static ip_address=192.168.1.152/24
static routers=192.168.1.1
static domain_name_servers=8.8.8.8 8.8.4.4
```

2) Static IP through router

### Port Forwarding


## Install necessary packages

Some of these may already be installed. Install on each node:

```
# Install latest version of docker
curl -sSL get.docker.com | sh && \
sudo usermod pi -aG docker
```

## Disable Swap

For Kubernetes 1.7 and later you will get an error if swap space is enabled. To turn it off:

```
sudo dphys-swapfile swapoff && \
sudo dphys-swapfile uninstall && \
sudo update-rc.d dphys-swapfile remove
```

Verify using the following, which should show no entries

```
sudo swapon --summary
```

## Enable cgroups

Necessary to avoid **CGROUPS_MEMORY: missing** error when running 'kubeadm init'

```
sudo nano /boot/cmdline.txt
```

Add in the entries cgroup_enable=cpuset cgroup_enable=memory which must appear before elevator=deadline, otherwise the changes won't take effect.

```
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=PARTUUID=1f8d51b7-02 rootfstype=ext4 cgroup_enable=cpuset cgroup_enable=memory elevator=deadline fsck.repair=yes rootwait
```

## Kubernetes

### What is it?

### Installation of Kubernetes on all nodes

Using 1.9.6 for now as per the guide, other versions had issues on the Raspberry Pi.

```
sudo apt update && sudo apt install -y apt-transport-https curl # enable installing from repo over https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - # add GPG key
sudo nano /etc/apt/sources.list.d/kubernetes.list
# add this line to the file: deb http://apt.kubernetes.io/ kubernetes-xenial main
sudo apt update
sudo apt install kubelet=1.9.6-00 kubeadm=1.9.6-00 kubectl=1.9.6-00
```

### Installation on the master node

```
# Experienced an error on one node for the following package being missing, fix was to run the following
apt install -y kubernetes-cni=0.6.0-00

# If you encontered issues and need to restart this process
sudo kubeadm reset
# Otherwise run init, ignoring the preflight error for the docker version
sudo kubeadm init --ignore-preflight-errors=SystemVerification

# Apply the recommended configuration output as part of the init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# If you miss the join command for the worker nodes, run the following on master
sudo kubeadm token create --print-join-command
```

### Install a network driver on master

There are a couple of options for network drivers including Weave and Flannel - here we will install Weave.

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.WEAVE_NO_FASTDP=1"
```

Why is it needed? - Without a network driver Kube DNS pods will never start.

### Verify that all the pods have started up

```
kubectl get pods --all-namespaces
```

### Installation on the worker nodes

The join command will be output on the master node with the current valid token, copy and paste into the worker nodes

```
sudo kubeadm join --ignore-preflight-errors=SystemVerification --token 224100.ac44e33db49cf25b 192.168.1.150:6443 --discovery-token-ca-cert-hash sha256:26e0e6df82d644eea1c115d1c8473dc4584a3dfc4ea24e14bd14c5d5e504eada
```

###

Verify that all the nodes are available

```
kubectl get nodes
```

### Useful Commands

```
kubectl get pods --all-namespaces
kubectl get deployments --all-namespaces
Then to delete the deployment:

kubectl delete -n NAMESPACE deployment DEPLOYMENT
kubectl delete -n kube-system deployment kubernetes-dashboard 

# To get the dashboard to start up (note we may need that command sudo sysctl -w net.ipv4.ip_forward=1 which I did run)
kubectl describe pod kubernetes-dashboard -n kube-system
kubectl scale deployment kubernetes-dashboard --replicas=0 -n kube-system
kubectl scale deployment kubernetes-dashboard --replicas=1 -n kube-system

kubectl describe pod kube-proxy -n kube-system
kubectl describe pod hello-world -n default

# To delete an installation which isn't working
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml

# To check if kube-proxy is running
ps auxw | grep kube-proxy

# If token expires, generate another
kubeadm token create

# To restart a failed pod, set the number of replicas to 0 and then back to 1
kubectl scale deployment kubia-xghm5 --replicas=0 -n default
kubectl scale deployment kubia-xghm5 --replicas=1 -n default

# To debug a failed pod
kubectl logs [podname] -p
kubectl logs hello-world -p

# On machines with systemd, the kubelet and container runtime write to journald.
# Errors reported by kubelet on master:
# Remember though you should run these command on the node running the pod
journalctl -u kubelet
journalctl -u kubelet | tail
journalctl -f

kubectl delete -n default deployment kubia-xghm5

# delete this one pod and DaemonSet will reschedule a new one
kubectl delete pod <podname>
kubectl delete pod kubia-xghm5
```

## Kubernetes Dashboard

### Installation

```
# Run the following on the master node
k3s kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml
k3s kubectl get pods --all-namespaces
kubectl cluster-info
```

### Accessing though the proxy

```
# Start the proxy to access the dashboard
k3s kubectl proxy
k3s kubectl proxy -p 8888

# From master node
curl localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

# Useful commands
k3s kubectl get po -n kube-system
k3s kubectl describe po -n kube-system <kubernetes-dashboard-pod-name>
k3s kubectl logs -n kube-system <kubernetes-dashboard-pod-name>
k3s kubectl logs -n kube-system kubernetes-dashboard-57df4db6b-scvt2
k3s kubectl delete pod <podname>
k3s kubectl delete pod kubernetes-dashboard-57df4db6b-scvt2
```

### Displaying the Dashboard from a Windows Machine on the Network


## Installing a basic Java application

### Accessing the application



## Pimorami LEDs

