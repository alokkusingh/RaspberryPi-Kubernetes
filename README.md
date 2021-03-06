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
### Add User/Group
Run the below using root user priviliage or sudo
	groupadd -g 600 singh
####
	useradd -u 601 -g 600 -s /usr/bin/bash alok
####
	mkdir /home/alok
####	
	chown -R alok:singh /home/alok/
####
	usermod -aG sudo alok
####
	passwd alok

### Set Timezone
	sudo timedatectl set-timezone Asia/Kolkata

### Install Docker
	sudo snap install docker
####
	sudo groupadd docker
####
	sudo usermod -a -G docker alok
####
	sudo mount -t ntfs-3g /dev/sdb1 /media/myexdic

### Install microk8s
	sudo snap install microk8s --classic
####
	sudo snap alias microk8s.kubectl kubectl
####
	sudo usermod -a -G microk8s alok
####
	sudo chown -f -R alok ~/.kube
####
	sudo nano /etc/docker/daemon.json

		{
		    "exec-opts": ["native, cgroupdriver=systemd"],
		    "log-driver": "json-file",
		     "log-opts": {
			"max-size": "100m"
		     },
		     "storage-driver": "overlay2"
		}
####
	sudo nano /etc/sysctl.conf

		net.ipv4.ip_forward=1
####
	kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

### Micro8s operation commnads
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
Kubernetes does not offer an implementation of network load balancers (Services of type LoadBalancer) for bare-metal clusters. If you???re not running Kubernetes on a supported IaaS platform (GCP, AWS, Azure???), LoadBalancers will remain in the ???pending??? state indefinitely when created.

Bare-metal cluster operators are left with two lesser tools to bring user traffic into their clusters, ???NodePort??? and ???externalIPs??? services.

For LoadBalancer service type we left with 2 choices:
###### Hard code the Node IPs - as below
	kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard

		spec:
		  type: LoadBalancer	# this has to be changed to LoadBalancer to access the dashboard externally 
		  externalIPs:
		  - 192.168.0.200		# this is needed because public IP cant be assigned automaic
###### Use one of the LoadBalancer implemented to olve this burpose. One of them is MetaLib.
MetalLB (https://metallb.universe.tf) aims to redress this imbalance by offering a network load balancer implementation that integrates with standard network equipment, so that external services on bare-metal clusters also ???just work??? as much as possible.

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
