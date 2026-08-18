# Set up K3s

Here’s how to set up and use K3s on a Debian computer.

## What is K3s?

K3s is a lighter version of Kubernetes that helps manage apps on computers and small devices. It's small, fast, and easy to use!

## What You Need:

A Debian Linux computer (or virtual machine) to run K3s.
An internet connection to download K3s.
Make sure your computer has no extra storage or unnecessary features.

Steps:

Step 1: Set Up Your Debian Computer

First, update your computer to make sure everything is up-to-date.
Open the terminal and type:
bash

```bash
sudo apt update && sudo apt upgrade -y
```

Restart your computer if needed.

Step 2: Turn Off Extra Features

K3s works best when there is no firewall or special security settings, so we’ll turn them off:

Turn off the firewall:
Type in the terminal:

```bash
sudo systemctl stop ufw

sudo systemctl disable ufw
```

Turn off SELinux (extra security features):
Edit the SELinux file by typing:

```bash
sudo nano /etc/selinux/config
```

Change SELINUX=enforcing to SELINUX=disabled, then save and exit by pressing Ctrl+X, then Y, and Enter.
Step 3: Set the Name for Your Computer

You need to set a name for your computer. Open the /etc/hosts file and add a name there.
Restart your computer so the changes take effect.
Step 4: Install Necessary Programs

Now, let’s install the required programs for K3s:

In the terminal, type:

```bash
sudo apt install -y curl lsof wget tar vim
```

Step 5: Install K3s

To install K3s, run this command:

```bash
curl -sfL https://get.k3s.io | sh
```

Step 6: Check If K3s Is Running

To check if K3s is working, type:

```bash
sudo systemctl status k3s
```

This will show you if everything is working fine.
Using K3s

Let’s try setting up a simple website with K3s!

Step 1: Create a File for Nginx Website

We will create a file to set up a simple Nginx server in K3s.

Create a file called nginx.yaml with the following content:

---
```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: nginx

  labels:

    app: nginx

spec:

  selector:

    matchLabels:

      app: nginx

  template:

    metadata:

      labels:

        app: nginx

    spec:

      containers:

      - name: nginx

        image: nginx:latest

        ports:

        - containerPort: 80
        
---

apiVersion: v1

kind: Service

metadata:

  name: nginx

  labels:

    app: nginx

spec:

  ports:

    - protocol: TCP

      port: 8081

      targetPort: 80

  selector:

    app: nginx

  type: LoadBalancer
```

Step 2: Run the Nginx Website

To create the Nginx server, type:

```bash
kubectl apply -f nginx.yaml
```

Check if the Nginx server is running:

```bash
kubectl get pods
```

Step 3: Visit Your Website

Look for the “EXTERNAL_IP” in the output and type it into your web browser with :8081 at the end.
For example: http://<EXTERNAL_IP>:8081

Step 4: Remove the Website

If you want to delete the website, type:

```bash
kubectl delete -f nginx.yaml
```

That’s it! Now you know how to set up K3s on a Debian computer and make a simple web server. 😊