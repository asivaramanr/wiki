# **Installing Kubernetes cluster on Azure Cloud VM's**

In this document we are going to install kubernetes on Azure VM's and will deploy the containers from there. You can use the [AzureCLI](https://github.com/asivaramanr/VisualStudio/tree/master/AzureCLI) & [Ansible](https://github.com/asivaramanr/VisualStudio/tree/master/Yaml/Ansible/Kubernetes_Install) scripts from my github  to proceed.

## Prerequisites:

* A Jumphost with PublicIP, (We will use jumphost as Ansible Master and Datahost for KubeMaster and Worker nodes)

* An SSH key pair from jumphost to all the servers. (Remember to enble root login to yes in sshd_config).

  Use the script `Azure_create_vm_jumphost.ps1` from [AzureCLI](https://github.com/asivaramanr/VisualStudio/tree/master/AzureCLI) to create a jumphost with PublicIP and in seperate VNET.

* Two or Three Linux servers with at least 2GB RAM and 2 vCPUs each. You should be able to SSH into each server as the root user with your SSH key pair.

  Use the script `Azure_create_vm_4kubernetes_NATGW.ps1` from [AzureCLI](https://github.com/asivaramanr/VisualStudio/tree/master/AzureCLI) for Kube Master and worker nodes.

  The above script will create the Data host with Debian 10 and in Seperate VNET from jumphost. 

* Ansible installed on jumphost. For installation instructions, follow the official [Ansible installation documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

Update the /etc/hosts file on all servers:
```
10.1.0.4 debian1 master
10.1.0.5 debian2 worker1
10.1.0.6 debian3 worker2
```
## Step 1 — Setting Up the Workspace Directory and Ansible Inventory File as root:

This will be on the jumphost we just created. 

```
mkdir ~/kube-cluster ; cd ~/kube-cluster
```
This directory will be our workspace and will contain all of our Ansible playbooks. It will also be the directory inside which you will run local commands.

Create a file named `~/kube-cluster/hosts` using nano or your favorite text editor and update below:

```
[masters]
master ansible_host=10.1.0.4 ansible_user=root

[workers]
worker1 ansible_host=10.1.0.5 ansible_user=root
worker2 ansible_host=10.1.0.6 ansible_user=root

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```
## Step 2 — Creating a Non-Root User (ansible) on All Remote Servers:

Use script `user_creation_initial.yml` from [Ansible](https://github.com/asivaramanr/VisualStudio/tree/master/Yaml/Ansible/Kubernetes_Install)

```
ansible-playbook -i hosts ~/kube-cluster/user_creation_initial.yml
```
## Step 3 — Installing Kubernetes Dependencies:

Use script `kube_dependencies_install.yml` from [Ansible](https://ggithub.com/asivaramanr/VisualStudio/tree/master/Yaml/Ansible/Kubernetes_Install)

```
ansible-playbook -i hosts ~/kube-cluster/kube_dependencies_install.yml
```
(Current version of k8s is 1.21.0-00 change it as required)

After execution, Docker, kubeadm, and kubelet will be installed on all of the remote servers. kubectl is not a required component and is only needed for executing cluster commands. 

## Step 4 — Setting Up the Master Node:

Use `kube-cluster/master_initial.yml` from [Ansible](https://github.com/asivaramanr/VisualStudio/tree/master/Yaml/Ansible/Kubernetes_Install)

```
ansible-playbook -i hosts ~/kube-cluster/master_initial.yml 
```

Once inside the master node as **ansible** user ssh ansible@master, execute below command:

```
kubectl get nodes
```
Output: 
```
ansible@debian1:~$ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
debian1   Ready    control-plane,master   20m   v1.21.0
ansible@debian1:~$
```
## Step 5 — Setting Up the Worker Nodes:

Use `kube-cluster/join_worker_nodes.yml` from [Ansible](https://github.com/asivaramanr/VisualStudio/tree/master/Yaml/Ansible/Kubernetes_Install)

```
ansible-playbook -i hosts ~/kube-cluster/join_worker_nodes.yml
```
## Step 6 — Verifying the Cluster:

```
kubectl get nodes
```

Output:
```
ansible@debian1:~$ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
debian1   Ready    control-plane,master   24m   v1.21.0
debian2   Ready    <none>                 81s   v1.21.0
debian3   Ready    <none>                 81s   v1.21.0
ansible@debian1:~$
```
If all of your nodes have the value Ready for STATUS, it means that they’re part of the cluster and ready to run workloads.

## Step 7 — Running An Application on the Cluster:

### Deployment

```
kubectl create deployment nginx --image=nginx
```
A deployment is a type of Kubernetes object that ensures there’s always a specified number of pods running based on a defined template, even if the pod crashes during the cluster’s lifetime. The above deployment will create a pod with one container from the Docker registry’s Nginx Docker Image.

Next, run the following command to create a service named nginx that will expose the app publicly. It will do so through a NodePort, a scheme that will make the pod accessible through an arbitrary port opened on each node of the cluster:

```
kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
```
### Services

Services are another type of Kubernetes object that expose cluster internal services to clients, both internal and external. They are also capable of load balancing requests to multiple pods, and are an integral component in Kubernetes, frequently interacting with other components.

Run the following command on maaster as ansible:
```
kubectl get services
```
Output:
```
ansible@debian1:~$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        34m
nginx        NodePort    10.104.93.128   <none>        80:32638/TCP   8m10s
ansible@debian1:~$
```
## To test that everything is working:

Run below command:
```
curl http://10.1.0.5:32638
```
Output:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
ansible@debian1:~$ 
```

!!! success
    You have successfully set up a Kubernetes cluster on Linux using Kubeadm and Ansible.

## Now Cleanup the resources tosave the bill.

### If you would like to remove the Nginx application, first delete the nginx service from the master node:
```
kubectl delete service nginx
```
### Run the following to ensure that the service has been deleted:
```
kubectl get services
```
Output:

```
ansible@debian1:~$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   38m
ansible@debian1:~$
```
### Then delete the deployment:
```
kubectl delete deployment nginx
```
### Run the `kubectl get services` again to confirm it worked:

Output:
```
"ansible@debian1:~$ kubectl get deployments
No resources found in default namespace.
ansible@debian1:~$"
```
 ### To delete all the resources created in Azure 

 We created all the resources in one single Resource Group. By deleting the RG, we will be deleting all the resources created for this expriment.
 
 Use `Azure_delete_resourcegroup.ps1 from` [AzureCLI](https://github.com/asivaramanr/VisualStudio/tree/master/AzureCLI)