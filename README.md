https://github.com/nothando12/assignment-1-Nothando.git

Name: Nothando Shandu

Student Number: 222482958

## Purpose
Deploy a lightweight Kubernetes cluster (k3s) on AWS, document the deployment, demonstrate evidence, and reflect on the work. This develops technical, problem-solving, documentation, and professional skills.


## System Requirements

| Component | Specification |
|-----------|--------------|
| Cloud Provider | AWS EC2 |
| Instance Type | t3.large |
| CPU | 2 vCPUs |
| RAM | 1 GB |
| Storage | 50 GB (EBS Volume) |
| Operating System | Ubuntu 24.04 
| Kubernetes Distribution | K3s |



# Installation Steps — K3s on AWS

These are the steps to deploy a Kubernetes (K3s) cluster on AWS.

## Step 1 — Connect to EC2 Instances
export PS1="k3s_server@ip-3.83.241.46:~$ "

export PS1="k3s_agent@ip-44.210.94.198:~$ "

## Update System Packages on instances
sudo apt update -y

sudo apt upgrade -y

sudo apt install -y curl
 
## Install K3s on the EC2 instances (control plane node):
curl -sfL https://get.k3s.io | sh -

sudo k3s kubectl get nodes

sudo k3s kubectl get pods -A

## successful install or helm deployment
mkdir -p ~/.kube

sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

sudo chown ubuntu:ubuntu ~/.kube/config

export KUBECONFIG=~/.kube/config

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version

helm repo add bitnami 

https://charts.bitnami.com/bitnami

helm repo update

helm install my-nginx bitnami/nginx

kubectl get deployments

## create a security group for your K3s cluster

export SG_ID=$(aws ec2 create-security-group \
  --group-name k3s-ha-sg \
  --description "K3s HA cluster security group" \
  --vpc-id $(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text --region $AWS_REGION) \
  --region $AWS_REGION \
  --query GroupId --output text)

echo 

## Find Latest Ubuntu 22.04 LTS AMI

export AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query "sort_by(Images,&CreationDate)[-1].ImageId" \
  --output text \
  --region $AWS_REGION)

echo

 ## Launch 3 EC2 Instances (Masters) for i in 1 2 3; do

 aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.large \
    --key-name $KEY_NAME \
    --security-group-ids $SG_ID \
    --subnet-id $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query Vpcs[0].VpcId --output text --region $AWS_REGION)" --query "Subnets[0].SubnetId" --output text --region $AWS_REGION) \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=k3s-master-$i}]" \
    --region $AWS_REGION \
    --query "Instances[0].InstanceId" --output text
done

 ## Step 2: Prepare All Nodes
Do this on all 3 master nodes (k3s-master-1, 2, 3) via SSH.

 ssh -i k3s_master_1.pem ubuntu@54.174.236.168

sudo apt update -y
sudo apt upgrade -y

sudo apt install -y curl wget vim git

sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

Required for container networking:

sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf

sudo tee /etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo hostnamectl set-hostname k3s-master-1 

TO GET TOKEN: sudo cat /var/lib/rancher/k3s/server/node-token
exit
ssh -i my-k3s-key.pem ubuntu@<PUBLIC IP MASTER 2>
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://<MASTER-1 PRIVATE IP>:6443 \ 
  --token <PASTE_TOKEN_HERE> \
  --node-ip=<MASTER-2 PRIVATE IP>
sudo systemctl status k3s

Install K3s *as the first server*:

bash id="k3s-master1"
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.30.8+k3s1" sh -s - server \
  --cluster-init \
  --server https://172.31.92.179:6443 \
  --tls-san 3.95.24.101

Check status:

bash id="check-k3s1"
sudo systemctl status k3s
sudo k3s kubectl get nodes

sudo hostnamectl set-hostname k3s-master-2

sudo apt-get update && sudo apt-get upgrade -y
sudo timedatectl set-timezone UTC

sudo tee -a /etc/hosts <<EOF
172.31.27.129 k3s-master-1
172.31.19.72  k3s-master-2
172.31.23.157 k3s-master-3
EOF

sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab



##  Architecture Explanation 

## What is k3s ?
k3s is a highly lightweight, fully certified Kubernetes distribution. It is designed to be a single binary that contains everything needed to run a cluster.

## why is it used?
Resource Efficiency: It uses about half the memory of "stock" Kubernetes (k8s), making it perfect for edge devices (like Raspberry Pis), IoT, and CI/CD environments.
Simplicity: It automates complex tasks like certificate management and simplifies installation into a single command.
Low Overhead: It removes "legacy" or "alpha" features and external cloud providers to stay lean.

### Key Components

* Control Plane- Managed by the Server node. It bundles the API Server, Scheduler, and Controller Manager into a single         process to save memory.

* Agents- These are the worker nodes. They run the k3s agent process, which connects back to the server to manage the actual workloads (Pods).

* Container Runtime- k3s defaults to containerd. It’s lightweight and industry-standard, replacing the bulkier Docker engine.

* CNI (Networking)- It comes pre-packaged with Flannel as the default CNI (Container Network Interface) for simple, out-of-the-box pod-to-pod communication.

* Ingress / Load Balancer- It includes Traefik as the default Ingress Controller and ServiceLB (a Klipper-based load balancer) to handle incoming external traffic.

* Storage Approach- By default, it uses a Local Storage Provisioner that allows pods to use space on the node's local disk. For the cluster database, it uses SQLite (instead of etcd) for single-node setups.




##  Evidence of Deployment
The following screenshots demonstrate the successful deployment of the K3s cluster on AWS.
## Master Deployment
![Pods](Screenshots/master-pods.png)

## EC2 Instance
![Instance](Screenshots/master-instance.png)
![Instance](Screenshots/EC2-instance.png)
![Instance](Screenshots/k3s_server-instance.png)
![Instance](Screenshots/k3s_agent-instance.png)

## Nodes
![Nodes](Screenshots/k3s-server-nodes.png)
![Nodes](Screenshots/k3s-agent-nodes.png)

## Pods
![Pods](Screenshots/k3s-server-pods.png)
![Pods](Screenshots/k3s-agent-pods.png)

## Deployment
![Deployment](Screenshots/k3s-server-deployment.png)
![Deployemnt](Screenshots/k3s-agent-deployment.png)



## Technical Reflection

Working on this project gave me invaluable practical experience setting up and running a lightweight Kubernetes cluster on AWS with K3s. I gained useful knowledge in cluster management, container orchestration, and cloud infrastructure configuration through this approach. My comprehension of how contemporary programs are deployed in scalable, distributed systems has improved as a result of setting up EC2 instances, defining security groups, and connecting nodes within a K3s cluster. My understanding of the connection between application orchestration and infrastructure provisioning has also improved as a result of this project.

Creating safe connections to EC2 instances with SSH keys and public IP addresses was one of the main difficulties I ran across. When I first tried to connect using private IPs, I got "connection timed out" messages. A deeper comprehension of AWS networking principles, such as the use of public IP addresses, appropriate security group configuration, and appropriate SSH key file permission settings, was necessary to solve issue. Additionally, I had trouble adding screenshots to the README because of my Windows system's file permissions. I was able to efficiently access and arrange the necessary files by modifying file ownership and permissions.

I ran into a number of technical problems during the cluster setup, such as connection disruptions and service outages. I was able to identify and fix configuration flaws, network problems, and token mismatches by studying system logs using tools like journalctl and systemctl status. This greatly enhanced my confidence in debugging actual system faults and sharpened my problem-solving skills.

This study also demonstrated the importance of K3s in contemporary cloud-native settings. K3s is a useful platform for comprehending high-availability and scalable systems since, despite its lightweight architecture, it integrates key Kubernetes components. This is especially true for new technologies like 5G, where cloud-native designs are essential.

The significance of K3s in modern cloud-native environments was further illustrated by this study. Despite its lightweight architecture, K3s combines essential Kubernetes components, making it a valuable platform for understanding high-availability and scalable systems. This is particularly true for emerging technologies like 5G, where cloud-native designs are crucial.








