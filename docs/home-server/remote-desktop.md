---
tags:
  - Ubuntu
  - Remote Desktop
---

![xrdp.png](../assets/images/xrdp.png)



# Remote Desktop

Accessing the server running Ubuntu Desktop via Remote Desktop is a user friendly way of interacting with our home server.

We can use a tool called `xrdp`.

## Install xrdp on Ubuntu

```bash
sudo apt-get update
sudo apt-get install xrdp
```

## Start the xrdp Service 

```bash
sudo service xrdp start 
```

## Allow xrdp through firewall 

```bash 
sudo ufw allow 3389
```

## Check xrdp Status 

```bash
sudo service xrdp status
```

## Connect to the servers IP Address via Windows Remote Desktop

To get the IP address of the Ubuntu server, run:

```bash
ip addr show
```