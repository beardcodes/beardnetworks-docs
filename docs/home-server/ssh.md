---
tags:
  - Ubuntu
  - SSH
---

![openssh.webp](../assets/images/openssh.webp)


# SSH

### Installing SSH is very straight forward using OpenSSH.

## Install and Enable OpenSSH

First let's install OpenSSH.

```bash
sudo apt update
sudo apt install openssh-server
```
Start the service

```bash
sudo service ssh start
```

Check the status to ensure it's running

```bash
sudo service ssh status
```

You should now see OpenSSH if you check your firewall app list

```bash
sudo ufw app list
```

You can then either allow port 22, or simply the OpenSSH, both will Open port 22. I like seeing the apps in the ufw status so lets go with the app name.

```bash
sudo ufw allow OpenSSH
```
You can now SSH into your home server! If you need to SSH from outside of your home network, you can either open port forwarding to your Servers IP address, or follow the documentation for setting up a VPN. If you are connected to your VPN service, you will be able to login to the server via SSH as well.

## Enable RSA Key Authentication

```bash
sudo nano /etc/ssh/sshd_config
```
Uncomment the following lines

```bash
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys .ssh/authorized_keys2
```
Save the file.

Restart the SSH service.

```bash
sudo systemctl restart ssh
```

Ensure proper permission are set.

```bash
sudo chown -R beardnetwork:bdeardnetwork ~/.ssh
sudo chmod 700 ~/.ssh
sudo chmod 600 ~/.ssh/authorized_keys
sudo chown -R beardnetwork:beardnetwork /home/beardnetwork
sudo chmod 700 /home/beardnetwork
```


!!! note "Replace Values"
<div class="grid cards" markdown>
Replace `beardnetwork` with the user name
</div>

## Remote Access of the network

### Prerequisites

You'll just want to make sure you've already set up a DDNS.

### Generate RSA key on Windows Client
On the windows machine you want to use to connect to your server via SSH, generate a key pair

```bash
ssh-keygen -t rsa -b 4096 "my-windows-client-key"
```
The default save path is fine.

Provide a strong passphrase.

Open the public key file and copy the content.

We're now going to add this public key to the server.

```bash
sudo nano ~/.ssh/authorized_keys
```
Paste the public key in here.

Save the file.

You can now connect to your server via SSH using the private key you generated on the client.

If you want to use WinSCP, use PuttyGen to convert the private key to a .ppk file.

### Disable Password Authentication

We definitely want to disable password authentication for security purposes now that we have the RSA key working.

```bash
sudo nano /etc/ssh/sshd_config
```
Uncomment the following lines and make sure the value is no

```bash
PasswordAuthentication no
```
Save the file.

Restart the SSH service.

```bash
sudo systemctl restart ssh
```

### Set up an Android Client
I downloaded ConnectBot which is a highly rated free SSH app for Android.

Go to Menu -> Manage Pubkeys

Create a new RSA key with a strong password as well as an appropriate nickname. After the key-pair is generated, you'll want to add the public key to the server like we did with the windows client.

Create a new host with your `<username>@<myDDNS>:22`, and you'll be able to connect as soon as you enable port forwarding on the router.

### Allow port forwarding
Make sure you've open port 22 on your router and forwarded it to your Server's IP address. You can always close it if you don't need to use SSH at the moment.

