# Helm on K3s

## Why Helm does not work on K3s

In my previous post, we have installed K3s on a new Linux machine.

However, when you install K3s by default, it only includes the kubectl command line tool, but not the helm tool.

Even if you install the Helm tool and try to deploy a chart, you most likely receive the following Error message:

```bash
Error: Kubernetes cluster unreachable
```

The reason is that Helm is not integrated with the K3s cluster, and even though it appears that it is working, it still cannot reach the server.

Setup Helm on K3s 

Below are the step-by-step instructions to install and configure Helm on K3s cluster using the easier approach.

Step 1: Install Helm

Install the helm from the official website. There are several ways to install it, but the most easiest way is to use the Script Installer.

📍 NOTE: Make sure that you have the curl and git commands installed on your machine.

Run the following commands to install the helm tool:

# download the helm script via curl command

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

# apply appropiate permissions
```bash
chmod 700 get_helm.sh
# run the script
./get_helm.sh
```

Step 2: Configure the Cluster Access Environment Variable

According to the K3s official documentation, for Helm to work correctly, you need to provide the Cluster Access via the KUBECONFIG Environment Variable, that is located at the path: /etc/rancher/k3s/k3s.yaml.

This KUBECONFIG Environment Variable will be looked by Helm as by default Helm have it defined as:

```bash
$KUBECONFIG set an alternative Kubernetes configuration file (default "~/.kube/config")
```

Run the following commands to configure it:

```bash
# export the yaml file
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
# get all pods
kubectl get pods --all-namespaces
# list all deployments via helm
helm ls --all-namespaces
```

If the helm command does provide the output, it means that the KUBECONFIG Environment Variable is configured correctly.

Helm working correctly on k3s kubernetes cluster

Testing the Helm Configuration

In order to make sure that helm is configured correctly, we can perform a test by using the WordPress Helm chart.

Step 1: Download the Helm Chart

Download the chart from the https://artifacthub.io/ website and search for WordPress.

Step 2: Install the Helm Chart

Run the following commands to install the chart:

```bash
# add the repo
helm repo add bitnami https://charts.bitnami.com/bitnami
# update the repo
helm repo update
# install the chart
helm install my-wordpress bitnami/wordpress
```

Step 3: Verify Chart Installation

Run the helm list command and it should show you the chart, that is named as my-wordpress.

This confirms that Helm has access to the K3s cluster and can access it resources.

Now you can simply uninstall this test chart via command:

```bash
helm delete my-wordpress
```