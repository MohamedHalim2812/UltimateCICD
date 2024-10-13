
# Infrastructure

Start by creating the infrastructure using Terraform on AWS environment. First we will run the terraform script to create the Kubernetes cluster (Master & 2 Worker Nodes) in addition to the Monitoring instance.


## Terraform

#### Providers
    terraform {
    required_providers {
        aws = {
        source  = "hashicorp/aws"
        version = "~> 5.0"  # Use the latest version that suits your needs
        }
    }
    }

    provider "aws" {
        region = "us-east-1"
        
    }

#### Use the default VPC:
    data "aws_vpc" "Default-VPC" { 
        default = true
    }


#### Generate a Key Pair and exporting the DEPI-KeyPair.pem file:
    resource "tls_private_key" "DEPI-Key" {  
        algorithm = "RSA"
        rsa_bits = 4096
    }

    # Create the key pair using the public key generated above
    resource "aws_key_pair" "DEPI-KeyPair" { 
        key_name = "DEPI-KeyPair"
        public_key = tls_private_key.DEPI-Key.public_key_openssh
    }

    # Create a local file to save the private key
    resource "local_file" "KeyPair" { 
        content = tls_private_key.DEPI-Key.private_key_pem
        filename = "DEPI-KeyPair.pem"
    }

    # Output the private key path
    output "private_key_path" { 
        value = local_file.KeyPair.filename
    }

#### Configure the Security Group for the instances:
    resource "aws_security_group" "DEPI-SecurityGroup" { 
        vpc_id = data.aws_vpc.Default-VPC.id
        name = "DEPI-SecurityGroup"
        ingress { #To access the VMs via SSH
            from_port = 22
            to_port = 22
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #For email notifications (will not be used in the project)
            from_port = 25
            to_port = 25
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #Range used for most of the applications
            from_port = 3000
            to_port = 10000
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #HTTP
            from_port = 80
            to_port = 80
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #HTTPS
            from_port = 443
            to_port = 443
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #Required when setting up Kubernetes cluster
            from_port = 6443
            to_port = 6443
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #Range used to send mail notification from our Jenkins pipeline to our gmail address
            from_port = 465
            to_port = 465
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        ingress { #Range used for deployment of applications
            from_port = 30000
            to_port = 32767
            protocol = "tcp"
            cidr_blocks = ["0.0.0.0/0"]
        }

        egress {
            from_port = 0
            to_port = 0
            protocol = "-1"
            cidr_blocks = ["0.0.0.0/0"]
        }
    
    }

#### Kubernetes Cluster

    # Create the EC2 instances
    resource "aws_instance" "Master" {
        ami           = "ami-0866a3c8686eaeeba"
        instance_type = "t2.micro"
        key_name = aws_key_pair.DEPI-KeyPair.key_name
        tags = {
            Name = "Master"
        }
    } 

    resource "aws_instance" "Slave-01" {
        ami           = "ami-0866a3c8686eaeeba"
        instance_type = "t2.micro"
        key_name = aws_key_pair.DEPI-KeyPair.key_name
        tags = {
            Name = "Slave-01"
        }
    } 

    resource "aws_instance" "Slave-02" {
        ami           = "ami-0866a3c8686eaeeba"
        instance_type = "t2.micro"
        key_name = aws_key_pair.DEPI-KeyPair.key_name
        tags = {
            Name = "Slave-02"
        }
    } 

#### Monitoring Server
    resource "aws_instance" "Monitoring" {
        ami           = "ami-0866a3c8686eaeeba"
        instance_type = "t2.micro"
        key_name = aws_key_pair.DEPI-KeyPair.key_name
        tags = {
            Name = "Monitoring"
        }
    } 
## Terraform apply


### 1. Use the default VPC
 

### 2. Key Pair
 

### 3. Security Group
 

### 4. EC2 instances created using Terraform
 

## Setup the Kubernetes Cluster

### 1. Access the instances using MobaXterm application.
    1.	Create a new session. 
    2.	Get the public IP address for each instance from AWS.	 
    3.	Copy the public IP address for each instance to the Remote host.
    4.	Check the Specify username box and enter “ubuntu” as the username.
    5.	In the Advanced SSH settings, check the Use private key box and place the .pem file.
    6.	Duplicate the session to create the 2 worker nodes and the Monitoring sessions as well by replacing the Remote host with each IP address.
    

### 2. Setup the Master and Worker Nodes
    1. Run the below command to change to root [On Master & Worker Node]

        • sudo su 

    2. Create an executable file and place the following commands then run the script [On Master & Worker Node]

        #	Update System Packages 
            •	sudo apt-get update

        #	Install Docker 
            •	sudo apt install docker.io -y
            •	sudo chmod 666 /var/run/docker.sock

        #	Install Required Dependencies for Kubernetes 
            •	sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
            •	sudo mkdir -p -m 755 /etc/apt/keyrings

        #	Add Kubernetes Repository and GPG Key 
            •	Curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg -- dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
            •	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

        #	Update Package List 
            •	sudo apt update

        #	Install Kubernetes Components
            •	sudo apt install -y kubeadm=1.28.1-1.1 kubelet=1.28.1-1.1 kubectl=1.28.1-1.1

    3. Run the following commands on the Master node only

        #   Initialize Kubernetes Master Node 
            •	sudo kubeadm init--pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all
        #   After running the above command then our vm will acts as master node and it will generate token to connect this with slave node-copy the token and run the command in slave machines 1 & 2

        #   Configure Kubernetes Cluster 
            •	mkdir -p $HOME/.kube
            •	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            •	sudo chown $(id -u):$(id -g) $HOME/.kube/config

        #   Deploy Networking Solution (Calico) 
            •	kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/calico.yaml

        #   Deploy Ingress Controller (NGINX) 
            •	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml

    4.  We'll Scan Kubernetes Cluster For Any Kind Of Issues Using Cube Audit

        # Go To The Website & Copy The Linux_amd_64 Link
            •	https://github.com/shopify/kubeaudit/releases
        # Paste It Using wget Command
        # Now Untar The File Using tar-xvf File Name
        # sudo mv kubeaudit /usr/local/bin/->kubeaudit all
