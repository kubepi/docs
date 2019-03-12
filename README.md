# Installation and setup instructions for a Kubernetes cluster on Raspberry Pi

## Distributions

Several distributions were trialled for use in this project. They include:

- Hypriot - Found it didn't work and is not maintained.
- Ubuntu-Mate
- Unbuntu Core - Minimal distro. Required to install a classic mode to access common features. Ran into multiple issues as a result of this, so moved on.
- Unbuntu (arm version)

Ubuntu was the version used in the final implementation. You can download it here.

## Ubuntu Configurations

### Static IP

There are 2 approaches that can be taken to set a static IP. For this project, since the intention is to allow it to work on multiple networks (home network along with a travel network i.e. different routers), the preference is to configure a static IP through Ubuntu itself. However, for future reference both methods will be documented.

1) Static IP through Ubuntu


2) Static IP through router

### Port Forwarding


### Configure a hostname for each device in the cluster


### Set an alias


### Run a start up script


# Temporary Activation of IP forwarding
sysctl -w net.ipv4.ip_forward=1
'''


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


