# RaspberryPi-Kubernetes
Kubernetes cluster setup on Raspberry Pi 4B

## Raspberry Pi Information
| Item | Information |
| :---: | :---: |
| Number of Pi Server | 2 |
| Pi1 Hostname | jgte - static IP assigned by router (192.168.0.200) |
| Pi2 Hostname | khbr - static IP assigned by router (192.168.0.201) |
| CPU Architecture | ARM64 (not AMD64) |
| Processor | 1.5GHz quad-core processors |
| RAM | 8GB |
| Disk | SanDisk MicroSD 32GB |
| OS | Linux jgte 5.4.0-1041-raspi #45-Ubuntu SMP PREEMPT Thu Jul 15 01:17:56 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux |

## Table of contents
<!-- TOC -->
* [RaspberryPi-Kubernetes](#raspberrypi-kubernetes)
  * [Raspberry Pi Information](#raspberry-pi-information)
  * [Table of contents](#table-of-contents)
  * [Raspberry Pi Setup](#raspberry-pi-setup)
    * [Prepare Boot Card](#prepare-boot-card)
  * [Dev OS update](#dev-os-update)
    * [Add host entry](#add-host-entry)
  * [OS Setup](#os-setup)
    * [First Time Login](#first-time-login)
    * [Enable Etehrnate interface with Static IP](#enable-etehrnate-interface-with-static-ip)
    * [Set Hostname](#set-hostname)
    * [Add User/Group](#add-usergroup)
    * [Add user to sudor group](#add-user-to-sudor-group)
    * [Create Password-less SSH Login](#create-password-less-ssh-login)
    * [Install Net Tools](#install-net-tools)
    * [Install Snap Package Manager](#install-snap-package-manager)
    * [OS Configs](#os-configs)
  * [Docker](#docker)
    * [Install Docker Daemon](#install-docker-daemon)
    * [Add user in group](#add-user-in-group)
    * [Update Config](#update-config)
  * [Microk8s](#microk8s)
    * [Preparation](#preparation)
    * [Installation](#installation)
    * [Add user in group](#add-user-in-group-1)
    * [Setup](#setup)
      * [Create `kubectl` alias](#create-kubectl-alias)
      * [Start Microk8s](#start-microk8s)
      * [Enable DNS](#enable-dns)
      * [Enable Ingress](#enable-ingress)
      * [Enable RBAC](#enable-rbac)
    * [Create Remote User - `alok`](#create-remote-user---alok)
      * [Create CSR for user `alok` and copy to Kubernetes master node](#create-csr-for-user-alok-and-copy-to-kubernetes-master-node)
      * [Sign User CSR on master node](#sign-user-csr-on-master-node)
      * [Copy Signed User Certificate to local server](#copy-signed-user-certificate-to-local-server)
      * [Copy CA Certificate to local server](#copy-ca-certificate-to-local-server)
      * [Create User Credentials - `alok`](#create-user-credentials---alok)
      * [Create Cluster - `home-cluster`](#create-cluster---home-cluster)
      * [Bind User `alok` Context to Cluster `home-cluster` - `alok-home`](#bind-user-alok-context-to-cluster-home-cluster---alok-home)
      * [Use the context - `alok-home`](#use-the-context---alok-home)
      * [Test command](#test-command)
    * [Micro8s operation commands](#micro8s-operation-commands)
  * [Kubernetes Setup](#kubernetes-setup)
    * [Node Cluster Setup](#node-cluster-setup)
    * [Setup Deployments - Kafka Cluster Setup](#setup-deployments---kafka-cluster-setup)
      * [Create Namespaces](#create-namespaces)
      * [Create Netwrok policy](#create-netwrok-policy)
      * [Create Zookeeper - Pod/Deployment/Service](#create-zookeeper---poddeploymentservice)

        * [In case want to delete](#in-case-want-to-delete)
      * [Create Kafka Broker - Pod/Deployment/Service](#create-kafka-broker---poddeploymentservice)

        * [In case want to delete](#in-case-want-to-delete-1)
      * [Setup Kubernetes Dashboard](#setup-kubernetes-dashboard)

        * [Change service type to LoadBalancer to access the dashboard externally](#change-service-type-to-loadbalancer-to-access-the-dashboard-externally)
        * [Work arround for configuring Netwrok Load Balancing](#work-arround-for-configuring-netwrok-load-balancing)
          * [Hard code the Node IPs - as below](#hard-code-the-node-ips---as-below)
          * [Use one of the LoadBalancer implemented to olve this burpose. One of them is MetaLib.](#use-one-of-the-loadbalancer-implemented-to-olve-this-burpose-one-of-them-is-metalib)
        * [Crete Service Account for Dashboard Login](#crete-service-account-for-dashboard-login)
        * [Assign Role to Service Account](#assign-role-to-service-account)
        * [Get Dashboard Login Admin Credential](#get-dashboard-login-admin-credential)

  * [Misc Commands](#misc-commands)





    * [](#)


















    * [](#-1)






    * [](#-2)



<!-- TOC -->

## Raspberry Pi Setup
### Prepare Boot Card
Ubuntu Server Boot setup
1. Choose Pi Version/OS/Storage
2. EDIT SETTINGS
   1. GENERAL 
      1. Set wireless LAN setting
      2. Set locals setting
      3. Enable SSH and User: aloksingh
```
https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#2-prepare-the-sd-card
```
---
## Dev OS update
### Add host entry
Add below entry in `/etc/hosts`
````
192.168.1.200   jgte kubernetes
````
```shell
vim /etc/hosts
```
---
## OS Setup
### First Time Login
xxx to be replaces by looking at the dynamic IP allocated for the Iefi interface
```shell
ssh aloksingh@192.168.1.xxx
```
### Enable Etehrnate interface with Static IP
Add below config in `/etc/netplan/50-cloud-init.yaml`. You may leave Wifi config as is.
```shell
network:
    version: 2
    renderer: networkd
    ethernets:
      eth0:
        dhcp4: no
        addresses:
          - 192.168.1.200/24
        nameservers:
          addresses: [8.8.8.8,8.8.8.4]
        routes:
          - to: default
            via: 192.168.1.1
```
```shell
sudo netplan apply
```
### Set Hostname
```shell
ssh aloksingh@jgte sudo -S hostnamectl set-hostname jgte 
```
### Add User/Group
```shell
ssh aloksingh@jgte sudo -S groupadd -g 600 singh
```
```shell
ssh aloksingh@jgte sudo -S useradd -u 601 -g 600 -s /usr/bin/bash alok
```
```shell
ssh aloksingh@jgte sudo -S mkdir /home/alok
```
```shell
ssh aloksingh@jgte sudo -S chown -R alok:singh /home/alok/
```
```shell
ssh aloksingh@jgte sudo -S passwd alok
```
---
### Add user to sudor group
```shell
ssh aloksingh@jgte sudo -S usermod -aG sudo alok 
```
---
### Create Password-less SSH Login
```shell
ssh-keygen
```
```shell
cat ~/.ssh/id_rsa.pub | ssh alok@jgte "mkdir -p ~/.ssh && cat >>  ~/.ssh/authorized_keys"
```
---
### Install Net Tools
```shell
ssh alok@jgte sudo -S apt install net-tools
```
---
### Install Snap Package Manager
```shell
ssh alok@jgte sudo -S apt install snapd
```
---
### OS Configs
Enable IP forward
````
net.ipv4.ip_forward=1
````
```shell
ssh alok@jgte sudo -S vim /etc/sysctl.conf
```
Set Timezone
```shell
ssh alok@jgte sudo timedatectl set-timezone Asia/Kolkata
```
---
## Docker
### Install Docker Daemon
```shell
ssh alok@jgte curl -fsSL https://get.docker.com -o get-docker.sh
```
```shell
ssh alok@jgte sh get-docker.sh
```
### Add user in group
```shell
ssh alok@jgte sudo -S groupadd docker
```
```shell
ssh alok@jgte sudo -S usermod -a -G docker alok
```
### Update Config
Add below in `/etc/docker/daemon.json`
````
{
    "exec-opts": ["native, cgroupdriver=systemd"],
    "log-driver": "json-file",
     "log-opts": {
        "max-size": "100m"
     },
     "storage-driver": "overlay2"
}
````
```shell
ssh alok@jgte sudo nano /etc/docker/daemon.json
```
---
## Microk8s
### Preparation
Add below line in file `/boot/firmware/cmdline.txt` (add in the same line from start)
````
cgroup_enable=memory cgroup_memory=1
````
```shell
ssh alok@jgte sudo -S nano /boot/firmware/cmdline.txt
```
---
### Installation
```shell
ssh alok@jgte sudo -S snap install microk8s --channel=1.25/stable --classic
```
### Add user in group
By adding user in microk8s group, user will have full access to the cluster 
```shell
ssh alok@jgte sudo -S usermod -a -G microk8s alok
```
```shell
ssh alok@jgte sudo -S chown -f -R alok ~/.kube
```
---
### Setup
#### Create `kubectl` alias
```shell
ssh alok@jgte sudo -S snap alias microk8s.kubectl kubectl
```
---
#### Start Microk8s
```shell
ssh alok@jgte -S microk8s.start
```
---
#### Enable DNS
```shell
ssh alok@jgte -S microk8s enable dns
```
-----
#### Enable Ingress
Enable Nginx Ingress Controller. This will deploy a daemonset nginx-ingress-microk8s-controller.
```shell
ssh alok@jgte -S microk8s enable ingress
```
-----
#### Enable RBAC
```shell
ssh alok@jgte microk8s enable rbac
```
---
### Create Remote User - `alok`
#### Create CSR for user `alok` and copy to Kubernetes master node
```shell
cd ~/cert/k8s
openssl genrsa -out alok.key 2048
openssl req -new -key alok.key -out alok-csr.pem -subj "/CN=alok/O=home-stack/O=ingress"
ssh alok@jgte mkdir cert
scp alok-csr.pem alok@jgte:cert/
```
#### Sign User CSR on master node
```shell
ssh alok@jgte "openssl x509 -req -in ~/cert/alok-csr.pem -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out ~/cert/alok-crt.pem -days 365"
```
#### Copy Signed User Certificate to local server
```shell
scp alok@jgte:cert/alok-crt.pem ~/cert/k8s
```
#### Copy CA Certificate to local server
```shell
scp alok@jgte:/var/snap/microk8s/current/certs/ca.crt ~/cert/k8s
```
#### Create User Credentials - `alok`
```shell
kubectl config set-credentials alok --client-certificate=/Users/aloksingh/cert/k8s/alok-crt.pem --client-key=/Users/aloksingh/cert/k8s/alok.key --embed-certs=true
```
---
#### Create Cluster - `home-cluster`
```shell
kubectl config set-cluster home-cluster --server=https://kubernetes:16443 --certificate-authority=/Users/aloksingh/cert/k8s/ca.crt --embed-certs=true
```
#### Bind User `alok` Context to Cluster `home-cluster` - `alok-home`
```shell
kubectl config set-context alok-home --cluster=home-cluster --namespace=home-stack --user alok
```
#### Use the context - `alok-home`
```shell
kubectl config use-context alok-home
```
---
#### Test command
Note: you will get permission denied as the role binding not done yet for the user `alok`
```shell
kubectl get nodes -o jsonpath='{}'
```
---

### Micro8s operation commands
| Command Description | Command |
| :---: | :--- |
| Start Kubernetes services| ``microk8s.start`` |
| The status of services| ``microk8s.inspect`` |
| Stop all Kubernetes services| ``microk8s.stop`` |
| Status of the cluster| ``microk8s.kubectl cluster-info`` |
| Set up DNS microk8s enable dns| ``microk8s enable dns`` |


## Kubernetes Setup
### Node Cluster Setup
| Command Description | Command |
| :---: | :--- |
| Add Master Node| ``sudo microk8s.add-node`` |
| Add a Slave Node| ``microk8s join 192.168.0.200:25000/201afbfc67544696d01eed22a56d5030/4496beb91a5d`` |
| Node List| ``kubectl get nodes`` |
| Remove Node by Master| ``sudo microk8s remove-node <node name>`` |
| Leave Node by Slave| ``sudo microk8s.leave`` |


### Setup Deployments - Kafka Cluster Setup
#### Create Namespaces
	kubectl apply -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/namespace-kafka.yaml

#### Create Netwrok policy
	kubectl apply -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/kafka-networkpolicy.yaml

#### Create Zookeeper - Pod/Deployment/Service
	kubectl apply --validate=true --dry-run=client --filename=https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/zookeeper-cluster.yaml
#####
	kubectl apply -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/zookeeper-cluster.yaml  --namespace=kafka-cluster
	
##### In case want to delete
	kubectl delete -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/zookeeper-cluster.yaml  --namespace=kafka-cluster

#### Create Kafka Broker - Pod/Deployment/Service
	kubectl apply --validate=true --dry-run=client --filename=https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/kafka-cluster.yaml 
#####
	kubectl apply -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/kafka-cluster.yaml  --namespace=kafka-cluster

##### In case want to delete
	kubectl delete -f https://github.com/alokkusingh/kafka-experimental/blob/master/yaml/kafka-cluster.yaml  --namespace=kafka-cluster

#### Setup Kubernetes Dashboard
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
#####
	kubectl get svc -n kubernetes-dashboard
##### Change service type to LoadBalancer to access the dashboard externally
	kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard

		spec:
		  type: LoadBalancer	# this has to be changed to LoadBalancer to access the dashboard externally 

##### Work arround for configuring Netwrok Load Balancing
Kubernetes does not offer an implementation of network load balancers (Services of type LoadBalancer) for bare-metal clusters. If you’re not running Kubernetes on a supported IaaS platform (GCP, AWS, Azure…), LoadBalancers will remain in the “pending” state indefinitely when created.

Bare-metal cluster operators are left with two lesser tools to bring user traffic into their clusters, “NodePort” and “externalIPs” services.

For LoadBalancer service type we left with 2 choices:
###### Hard code the Node IPs - as below
	kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard

		spec:
		  type: LoadBalancer	# this has to be changed to LoadBalancer to access the dashboard externally 
		  externalIPs:
		  - 192.168.0.200		# this is needed because public IP cant be assigned automaic
###### Use one of the LoadBalancer implemented to olve this burpose. One of them is MetaLib.
MetalLB (https://metallb.universe.tf) aims to redress this imbalance by offering a network load balancer implementation that integrates with standard network equipment, so that external services on bare-metal clusters also “just work” as much as possible.

##### Crete Service Account for Dashboard Login
	kubectl create serviceaccount dashboard -n kafka-cluster
	
##### Assign Role to Service Account
	kubectl create clusterrolebinding dashboard-admin -n kafka-cluster --clusterrole=cluster-admin --serviceaccount=kafka-cluster:dashboard

##### Get Dashboard Login Admin Credential
	kubectl get secrets -n kafka-cluster
#####
	kubectl get secret dashboard-token-bspqw -n kafka-cluster -o jsonpath="{.data.token}" | base64 --decode

https://jgte

![alt text](https://github.com/alokkusingh/RaspberryPi-Kubernetes/blob/master/media/k-dbd-rs.png?raw=true "Kubernetes Dashboard")


## Misc Commands
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#port-forward

	kubectl api-resources
###
	kubectl explain pods
###
	kubectl describe secret
###
	sudo microk8s inspect
###
	kubectl get all --namespace=kafka-cluster
###
	kubectl get namespaces
###	
	kubectl get networkpolicy --namespace=kafka-cluster
###
	kubectl top node
###
	kubectl top pod --namespace=kafka-cluster
###
	kubectl apply --validate=true --dry-run=client --filename=yaml/zookeeper-cluster.yaml 
###
	kubectl apply -f yaml/zookeeper-cluster.yaml  --namespace=kafka-cluster
###
	kubectl delete -f yaml/zookeeper-cluster.yaml  --namespace=kafka-cluster
###
	kubectl get all --namespace=kafka-cluster
###
	kubectl get events --namespace=kafka-cluster
###
	kubectl describe pod pod/zookeeper-deployment-65487b964b-ls6cg
###
	kubectl get pod/zookeeper-deployment-65487b964b-ls6cg -o yaml --namespace=kafka-cluster
###
	kubectl logs pod/zookeeper-deployment-7549748b46-x65fp zookeeper --namespace=kafka-cluster
###
	kubectl logs --previous pod/zookeeper-deployment-7549748b46-jvvqk zookeeper --namespace=kafka-cluster
###
	kubectl describe pod/zookeeper-deployment-65cb748c5c-fv545 --namespace=kafka-cluster
###
	kubectl exec -it pod/zookeeper-deployment-7549748b46-9n9kb bash --namespace=kafka-cluster
	apt-get update
	apt-get install iputils-ping
	apt-get install net-tools
###
	kubectl apply --validate=true --dry-run=client --filename=yaml/kafka-cluster.yaml 
###
	kubectl apply -f yaml/kafka-cluster.yaml  --namespace=kafka-cluster
###
	kubectl delete -f yaml/kafka-cluster.yaml  --namespace=kafka-cluster
###
	kubectl logs pod/kafka-b8bdd7bc8-w2qq4 kafka --namespace=kafka-cluster
###
	kubectl exec -it pod/kafka-deployment-86559574cc-jpxwq bash --namespace=kafka-cluster
###	
	apt-get update
###
	kubectl cluster-info dump --namespace=kafka-cluster
###
	kubectl apply --validate=true --dry-run=client --filename=yaml/nginx-cluster.yaml
###
	kubectl apply -f yaml/nginx-cluster.yaml  --namespace=kafka-cluster
###
	kubectl delete -f yaml/nginx-cluster.yaml --namespace=kafka-cluster
###
	kubectl logs pod/nginx-deployment-6d9d878b78-tst2b nginx --namespace=kafka-cluster
###
	kubectl exec -it pod/nginx-6d9d878b78-kqlfb bash --namespace=kafka-cluster
###	
	nc -vz zookeeper-service 2181 -w 10
###
	kubectl get ep nginx-service --namespace=kafka-cluster
###
	kubectl describe svc nginx-service --namespace=kafka-cluster
###
	kubectl get pods --show-labels --namespace=kafka-cluster
