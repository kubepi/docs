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

Ubuntu Server was the version used in the final implementation. You can download it [here](https://wiki.ubuntu.com/ARM/RaspberryPi#arm64): 

**Raspberry Pi 3B/3B+: ubuntu-18.04.2-preinstalled-server-arm64+raspi3.img.xz**

## Ubuntu Configurations

### Static IP

There are 2 approaches that can be taken to set a static IP. For this project, since the intention is to allow it to work on multiple networks (home network along with a travel network i.e. different routers), the preference is to configure a static IP through Ubuntu itself. However, for future reference both methods will be documented.

1) Static IP through Ubuntu Server

Ubuntu Server 18.04 requires editing a .yaml file over the traditional updates into /etc/network/interfaces since it now uses Netplan.

*Netplan is a command line utility for the configuration of networking on certain Linux distributions. Netplan uses YAML description files to configure network interfaces and, from those descriptions, will generate the necessary configuration options for any given renderer tool.*

```
cd /etc/netplan
ls -l
# You should see a file named 50-cloud-init.yaml
# If you don't also see a file named 01-netcfg.yaml, create it with the command sudo touch 01-netcfg.yaml

# Now identify the network interface you want the changes applied to, i.e your ethernet port
ip a

# Apply the configurations
sudo nano 01-netcfg.yaml 
# Add in the yaml below for each node ...

# Restart netplan
sudo netplan apply

# If it fails run
sudo netplan --debug apply
```

Configure the master node with the following, assigning an IP of 192.168.1.150 (note that eth0 below should be replaced with the network interface identified when you ran **ip a** - in the output below, you can see that the interface is called eth0):

```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether b8:27:eb:11:16:e7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.151/24 brd 192.168.1.255 scope global dynamic eth0
       valid_lft 84632sec preferred_lft 84632sec
    inet6 fe80::ba27:ebff:fe11:16e7/64 scope link 
       valid_lft forever preferred_lft forever
3: wlan0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether b8:27:eb:44:43:b2 brd ff:ff:ff:ff:ff:ff
```

Configure the master node as follows:

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eth0:
            dhcp4: no
            addresses: [192.168.1.150/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [8.8.4.4,8.8.8.8]
```

Configure the worker nodes as follows:

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eth0:
            dhcp4: no
            addresses: [192.168.1.151/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [8.8.4.4,8.8.8.8]
```

And the final worker node:

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eth0:
            dhcp4: no
            addresses: [192.168.1.152/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [8.8.4.4,8.8.8.8]
```

2) Static IP through router

### Port Forwarding


### Configure a hostname for each device in the cluster

To help identify which node you are working on from the terminal, assign each node a unique hostname.

```
# Display the current hostname
hostnamectl

# Set the hostname for kuber-master
sudo hostnamectl set-hostname kuber-master

# Edit /etc/hosts and update the hostname
sudo nano /etc/hosts

# Edit cloud.cfg - first check if it exists
ls -l /etc/cloud/cloud.cfg
# If it doesn't, then you are done, otherwise complete the next steps
sudo nano /etc/cloud/cloud.cfg
# Search for preserve_hostname and set it to true, preventing the set+update hostname module from resetting it

# Reboot
sudo reboot
# Verify
hostnamectl
```

### Set an alias


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

## k3s (Lightweight Kubernetes)

### What is it?

### Installation on the master node

### Installation on the worker nodes


## Kubernetes Dashboard

### Installation

### Accessing though the proxy

### Displaying the Dashboard from a Windows Machine on the Network


## Installing a basic Java application

### Accessing the application



## Pimorami LEDs


