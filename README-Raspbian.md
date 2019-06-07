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

## Installing a basic HelloWorld Java/Spring application

As a simple first example (available [here](https://github.com/kubepi/java-spring-hello-world)) we will create a basic hello world application which will have the following requirements:

1) A simple RESTful API build in Java and Spring comprising of a single GET service to return a string
2) It should contain a docker image with a compatible base image
3) It should contain a kubernetes deployment descriptor to deploy the application onto Kubernetes

### Balena base images

The base image that is used must be compatible with the underlying architecture that the cluster will reside on. In this case, although the Raspberry Pi 3 B+ is 64bit (arm64) compatible with an appropriate distro installation, this cluster will use Raspbian, which in it's current form supports 32bit armhf.

A [Balena](https://www.balena.io/docs/reference/base-images/base-images/) has been used for this example which supports Java 8 on a 32bit armhf architecture.

### Installing the application

To install the application we apply the deployment.yaml we build for this example.

```
kubectl apply -f https://raw.githubusercontent.com/kubepi/java-spring-hello-world/master/deployment.yaml

# To delete the installation again, if you need to, run the following command
kubectl delete -f https://raw.githubusercontent.com/kubepi/java-spring-hello-world/master/deployment.yaml

# Verify the state of the installation using the following
kubectl describe pod hello-world -n default

# Note that initial install will take some time while the image is being pulled

# If you are having issues with the installation, you can debug through docker using the following command, which will open an interactive shell container
docker run -it bingbo/kubepi-helloworld:0.1.2 sh
```

### Accessing the application

There are several ways to access an application on the cluster, a good overview of which is available [here](https://gardener.cloud/050-tutorials/content/howto/service-access/). The primary methods used include:

- ClusterIP
- NodePort
- LoadBalancer

In the deployment.yaml used in this example, we have defined external access using NodePort.

A service of type NodePort is a ClusterIP service with an additional capability: it is reachable at the IP address of the node as well as at the assigned cluster IP on the services network.

Note that Kubernetes does not offer an implementation of network load-balancers (Services of type LoadBalancer) for bare metal clusters - typically this is achieved through a Load Balancer made available via a cloud provider.


```
kubectl get services

# If you need to delete the service for any reason
kubectl delete service hello-world-service

# To access the cluster using NodePort, first find out the Node IP address
kubectl describe node

Addresses:
  InternalIP:  192.168.1.152
  Hostname:    kuber-worker-b

# Then get the assigned port
kubectl get services

pi@kuber-master:~ $ kubectl get services
NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
hello-world-service   NodePort    10.99.9.14   <none>        80:31001/TCP   22h
kubernetes            ClusterIP   10.96.0.1    <none>        443/TCP        22d

kubectl exec -it hello-world-64484d76f9-fgqsp curl 192.168.1.152:31001
OR simply ...
curl 192.168.1.152:31001

# Output
Hello Kubernetes World!

# Scaling the service
kubectl get pods -o wide
kubectl scale deployment hello-world --replicas=2 -n default

# You should now see that the hello-world service is now available on both of our workers

pi@kuber-master:~ $ kubectl get pods -o wide
NAME                           READY     STATUS              RESTARTS   AGE       IP          NODE
hello-world-64484d76f9-dlnxm   0/1       ContainerCreating   0          4s        <none>      kuber-worker-b
hello-world-64484d76f9-fgqsp   1/1       Running             0          1d        10.44.0.3   kuber-worker-a

# And eventually an IP will be assigned
pi@kuber-master:~ $ kubectl get pods -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP          NODE
hello-world-64484d76f9-dlnxm   1/1       Running   0          1m        10.36.0.1   kuber-worker-b
hello-world-64484d76f9-fgqsp   1/1       Running   0          1d        10.44.0.3   kuber-worker-a
```

## Pimorami LEDs

See [here](https://github.com/apprenda/blinkt-k8s-controller) for details for how to install.

I also had to apply the following commands:

```
kubectl label node kuber-master deviceType=blinkt
kubectl label node kuber-worker-a deviceType=blinkt
kubectl label node kuber-worker-b deviceType=blinkt
```

## Load Balance test for NodeType

```
#!/bin/bash
> load_balance_results
> node_port_results
for i in {1..200}; 
do
  curl $1 >> load_balance_results
done
grep -hr "Pod Name:" load_balance_results > node_port_results
echo '******************************************************'
sort node_port_results | uniq -c
echo '******************************************************' 
```

Then run:

```
bash load_balance_test_node_port.sh 192.168.1.152:30123
```
