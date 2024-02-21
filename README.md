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

## Raspberry Pi Setup
### Prepare Boot Card
Ubuntu Server Boot setup
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
### Add User/Group
```shell
ssh ubuntu@jgte sudo groupadd -g 600 singh
```
```shell
ssh ubuntu@jgte sudo ueradd -u 601 -g 600 -s /usr/bin/bash alok
```
```shell
ssh ubuntu@jgte sudo mkdir /home/alok
```
```shell
ssh ubuntu@jgte sudo chown -R alok:singh /home/alok/
```
```shell
ssh ubuntu@jgte sudo usermod -aG sudo alok
```
```shell
ssh ubuntu@jgte sudo passwd alok
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
### Add user to sudor group
```shell
ssh ubuntu@jgte sudo usermod -aG sudo alok 
```
---
### Install Net Tools
```shell
ssh alok@jgte sudo apt install net-tools
```
---
### Install Snap Package Manager
```shell
ssh alok@jgte sudo apt install snapd
```
---
### OS Configs
Enable IP forward
````
net.ipv4.ip_forward=1
````
```shell
ssh alok@jgte sudo nano /etc/sysctl.conf
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
ssh ubuntu@jgte sudo groupadd docker
```
```shell
ssh ubuntu@jgte sudo usermod -a -G docker alok
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
ssh alok@jgte sudo nano /boot/firmware/cmdline.txt
```
---
### Installation
```shell
ssh alok@jgte sudo snap install microk8s --channel=1.25/stable --classic
```
### Add user in group
By adding user in microk8s group, user will have full access to the cluster 
```shell
ssh alok@jgte sudo usermod -a -G microk8s alok
```
```shell
ssh alok@jgte sudo chown -f -R alok ~/.kube
```
---
### Setup
#### Create `kubectl` alias
```shell
ssh alok@jgte sudo snap alias microk8s.kubectl kubectl
```
---
#### Start Microk8s
```shell
ssh alok@jgte microk8s.start
```
---
#### Enable DNS
```shell
ssh alok@jgte micrk8s enable dns
```
-----
#### Enable Ingress
```shell
ssh alok@jgte microk8s enable ingress
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
scp alok-csr.pem alok@jgte:cert/
```
#### Sign User CSR on master node
```shell
ssh alok@jgte openssl x509 -req -in ~/cert/alok-csr.pem -CA /var/snap/microk8s/current/certs/ca.crt -CAkey /var/snap/microk8s/current/certs/ca.key -CAcreateserial -out ~/cert/alok-crt.pem -days 365
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
