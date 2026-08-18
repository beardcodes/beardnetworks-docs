---
tags:
  - Ubuntu
  - Samba
---

# Mapped Network drive

Allowing yourself to map folders of your server as network drive makes it very easy to read and write to your server from another computer over your home network.

To accomplish this, we can use a service called samba.

## Step 1: Install Samba

```bash
sudo apt-get update
sudo apt-get install samba
```
## Step 2: Share a particular folder

```bash
mkdir -p /srv/shared
sudo chown -R :<group> /srv/shared
chmod -R 770 /srv/shared
```
!!! Note

You don't need to run `mkdir` if the folder already exists. The chmod command does the following: chmod: Stands for "change mode," a command in Unix and Unix-like operating systems to change the access permissions of files and directories. -R: Recursively changes the permissions of the specified directory and its contents. 770: Assigns read (4), write (2), and execute (1) permissions to the owner and group. In total, it grants full read, write, and execute permissions to the users and group.

!!! Tip

To create a linux user


```bash
sudo adduser <username>
To create a group
```
To create a group

```bash
sudo groupadd <group>
To add a user to a group
```
To add a user to a group

```bash
sudo usermod -aG $GROUP $USER
```
## Step 3: Configure Samba
Edit the Sasmba configuration file.

```bash
sudo nano /etc/samba/smb.conf
```
Add this at the end of the file:

```bash
[SharedFolder]
  path = /home/your_username/SharedFolder
  read only = no
  guest ok = yes
```

!!! Tip

Setting `guest ok = no` will make it such that a valid Samba user will need to provide their credentials to access the mapped drive.

## Step 4: Restart Samba

```bash
sudo service smbd restart
```
## Step 5: Create a Samba User
This user needs to be an existing Ubuntu user:
```bash
sudo smbpasswd -a your_samba_username
```

## Step 6: Allow access through firewall
If firewall is enabled on the Ubuntu machine, allow traffic for the Samba app.

```bash
sudo ufw allow Samba
```

!!! Info

To check the firewall status run

```bash
sudo ufw status
```
To activate the firewall run

```bash
sudo ufw enable
```
To view available applications you can enable

```bash
sudo ufw app list
```
## Step 7: Access from Windows

You should now be able to open an explorer instance on a Windows machine on the network and access \\your_ubuntu_ip\SharedFolder

!!! Tip

To find your Ubuntus ip address, run

```bash
ip addr show
```