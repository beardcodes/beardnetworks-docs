---
tags:
  - Ubuntu
  - Docker
---


![docker.png](../assets/images/docker.png)

# Docker

 Docker is very useful, particular for self hosting applications in their own environments, i.e. "containerizing". This is also highly recommended for an application like Immich, which is a phenomal Google Photos replacement.

## Docker and Docker Compose

Because I am using Ubuntu, I will follow the respective documentation here, [https://docs.docker.com/engine/install/ubuntu/]. The documentation provides two main methods, downloading Docker Desktop for Linux, and installing from the apt Repository. I decided to give the Docker Desktop a go.

## Docker Desktop
## KVM Setup
Docker Desktop requires kvm support

Load the module.

```bash
modprobe kvm
modprobe kvm_intel
```
Check if they are enabled.

```bash
lsmod | grep kvm
```
Add your user to the kvm group.

```bash
sudo usermod -aG kvm $USER
```

## Set up Docker's package repository
You do have to follow the first set of the apt method anyways. Start with

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
## Install DEB package

Download the DEB packager per https://docs.docker.com/desktop/install/ubuntu/#install-docker-desktop.

This went my downloads folder so to install it, I ran the following

```bash
sudo apt-get update
sudo apt-get install ./Downloads/docker-desktop-<version>-<arch>.deb
```
!!! note  "Fill in the place holders" 
This will be in the file name of the DEB package you downloaded.

 Test the Docker version

```bash
docker --version
```
Test the Docker Compose Version

```bash
docker-compose --version
```
## Troubleshooting Docker Compose

For some reason, Docker Compose was not recognized for me. To solve this, I followed the next steps on the apt repository guide

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Test the Docker Compose Version again.

```bash
docker-compose --version
```
I still didn't seem to have it downloaded, so I juust manually downloaded it this time.

```bash
sudo apt install docker-compose
```

And there we go. You should now be able to launch the Docker Desktop application.

!!! tip "kvm permission error" 

If you get an error regarding user permission to `/dev/kvm`. Make sure permissions are set correctly.
```bash
sudo chown root:kvm /dev/kvm
```
Give your user account read/write access.
```bash
sudo chmod 0666 /dev/kvm
```
Restart Docker.
```bash
sudo systemctl restart docker
```



