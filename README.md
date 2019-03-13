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

Configure the master node with the following, assigning an IP of 192.168.1.150 (note that ens5 below should be replaced with the network interface identified when you ran **ip a**):

```
network:
    version: 2
    renderer: networkd
    ethernets:
        ens5:
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
        ens5:
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
        ens5:
            dhcp4: no
            addresses: [192.168.1.150/24]
            gateway4: 192.168.1.1
            nameservers:
                addresses: [8.8.4.4,8.8.8.8]
```

2) Static IP through router

### Port Forwarding


### Configure a hostname for each device in the cluster


### Set an alias


### Run a start up script

```
# Temporary Activation of IP forwarding
sysctl -w net.ipv4.ip_forward=1
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


