# Installation and setup instructions for a Kubernetes cluster on Raspberry Pi

## Fundamentals

### Networking

- Understanding NAT

Linux in Action covers this well

## Distributions

Several distributions were trialled for use in this project. They include:

- Hypriot - Found it didn't work and is not maintained.
- Unbuntu Core - Minimal distro. Required to install a classic mode to access common features. Ran into multiple issues as a result of this that weren't worth the pain for what I wanted to do.
- Ubuntu-Mate - Worked fine, but difficulties in updating distribution from 16.04 to 18.04
- Unbuntu Server (ARM64 version)

## Raspian Configurations

### Enable SSH

```
Enter sudo raspi-config in a terminal window.
Select Interfacing Options.
Navigate to and select SSH.
Choose Yes.
Select Ok.
Choose Finish.
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


### Configure a hostname for each device in the cluster

To help identify which node you are working on from the terminal, assign each node a unique hostname.

```

```

### Set an alias

Note: Only complete this step after k3s has been installed

Since we are using k3s to run the kubernetes cluster (see later instructions), we can create aliases to save on typing in the k3s command each time.

```
# Open .bashrc for editing (options – B backups up the file, u is undo)
sudo nano -Bu ~/.bashrc

# Scroll down to the bottom of the file and add the desired aliases

# Custom Aliases
alias kubectl='k3s kubectl'
```

### Run a start up script

Create a script in each node in the home directory. Execute the following commands to add a reboot entry to crontab:

```
cd ~
sudo nano startup-tasks.sh
# Add the tasks to the file and save

# Make it executable
sudo chmod a+x startup-tasks.sh
# Give it root rights (saves you to write sudo every time)
sudo chmod 777 startup-tasks.sh

# Add a @reboot entry to crontab
crontab -e
```

crontab file should look like:

```
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
@reboot sudo ~/startup-tasks.sh
```

Add the following to startup-tasks.sh:

```
#! /bin/bash

# Temporary Activation of IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
# Switch off firewall and check status
sudo ufw disable
```

Reboot the PI and confirm that the script ran on startup:

```
# Should have a value of 1
sysctl net.ipv4.ip_forward
# Will show the status of the firewall, should be disabled
sudo ufw status
```

## Install necessary packages

Some of these may already be installed. Install on each node:

```
sudo apt-get install git
sudo apt-get install docker
```

## Kubernetes

### What is it?

### Installation on the master node

```

```

### Installation on the worker nodes

```

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

