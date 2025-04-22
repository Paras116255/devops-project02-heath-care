Heath care Domain Project



Tools:
-	Git - For version control for tracking changes in the code files 
-	 Jenkins - For continuous integration and continuous deployment 
-	 Docker - For containerizing applications 
-	Ansible - Configuration management tools 
-	Selenium - For automating tests on the deployed web application 
-	Terraform - For creation of infrastructure. 
-	Kubernetes – for running containerized application in managed cluster.


1.	Install terraform and aws cli
# sudo su -

# wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# sudo apt update && sudo apt install terraform -y

# apt-get update

# apt-get install awscli -y

# aws configure

 


# mkdir myfiles

# cd myfiles 

# vim main.tf
provider "aws" {

region = "us-east-1"

}

resource "aws_vpc" "paras-vpc" {
 cidr_block = "10.0.0.0/16"
  tags = {
   Name = "paras-vpc"
}
}

resource "aws_subnet" "subnet-1"{

vpc_id = aws_vpc.paras-vpc.id
cidr_block = "10.0.1.0/24"
depends_on = [aws_vpc.paras-vpc]
map_public_ip_on_launch = true
  tags = {
   Name = "paras-subnet"
}

}

resource "aws_route_table" "paras-route-table"{
vpc_id = aws_vpc.paras-vpc.id
  tags = {
   Name = "paras-route-table"
}

}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet-1.id
  route_table_id = aws_route_table.paras-route-table.id
}

resource "aws_internet_gateway" "gw" {
 vpc_id = aws_vpc.paras-vpc.id
 depends_on = [aws_vpc.paras-vpc]
   tags = {
   Name = "paras-gw"
}

}

resource "aws_route" "paras-route" {

route_table_id = aws_route_table.paras-route-table.id
destination_cidr_block = "0.0.0.0/0"
gateway_id = aws_internet_gateway.gw.id

}

variable "sg_ports" {
type = list(number)
default = [8080,80,22,443]

}


resource "aws_security_group" "paras-sg" {
  name        = "sg_rule"
  vpc_id = aws_vpc.paras-vpc.id
  dynamic  "ingress" {
    for_each = var.sg_ports
    iterator = port
    content{
    from_port        = port.value
    to_port          = port.value
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    }
  }
egress {

    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]

}
}

resource "aws_instance" "myec2" {
  ami           = "ami-0f9de6e2d2f067fca"
  instance_type = "t2.medium"
  key_name = "project-bnf"
  subnet_id = aws_subnet.subnet-1.id
  security_groups = [aws_security_group.paras-sg.id]
  tags = {
    Name = "Project-EC2-DEV"
  }

}

resource "aws_instance" "myec2-test" {
  ami           = "ami-0f9de6e2d2f067fca"
  instance_type = "t2.micro"
  key_name = "project-bnf"
  subnet_id = aws_subnet.subnet-1.id
  security_groups = [aws_security_group.paras-sg.id]
  tags = {
    Name = "Project-EC2-test"
  }

}


# terraform init 

# terraform apply --auto-approve





2.	Now connect to the Dev Server which got created on AWS
Install Ansible
1  sudo apt update
2  sudo apt install software-properties-common
3  sudo add-apt-repository --yes --update ppa:ansible/ansible
4  sudo apt install ansible –y

RUN THE JENKINS KEY commands before you run the playbook

# sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
# vim playbook-install.yml
- name: Install and setup devops tools
  hosts: localhost
  become: true
  tasks:
    - name: Update the apt repo
      command: apt-get update
    - name: Install multiple packages
      package: name={{item}} state=present
      loop:
        - git
        - docker.io
        - openjdk-17-jdk
    - name: install jenkins
      command: sudo apt-get install jenkins -y
    - name: start jenkins and docker service
      service: name={{item}} state=started
      loop:
        - jenkins
        - docker

 
 

Go to Ansible COntroller --> copy ssh keys of root user on the test server 

Create the inventory file and ansible.cfg file 

# vim myinventory 

[webserver]
 10.0.1.110 
# vim ansible.cfg 
[defaults]
inventory = /root/myinventory

# vim playbook-test-server.yml
- name: Install and set up test server
  hosts: webserver
  become: true
  tasks:
    - name: Update the apt repo
      command: apt-get update
    - name: Install multiple packages
      package: name={{item}} state=present
      loop:
        - git
        - docker.io
        - openjdk-17-jdk
    - name: start Jenkins and docker service
      service: name={{item}} state=started
      loop:
        - docker


save and run the playbook 

Go to test server

# git --version 

# java -version 

create jenkins root directory 

# cd /tmp 

# mkdir jenkinsdir 

# chmod -R 777 /tmp/jenkinsdir 

#  chmod -R 777 /var/run/docker.sock

Connect test server to Jenkins as agent 

Create a node on jenkins server and continue the pipleine


Pipeline code
pipeline{
    agent none
    tools{
        maven 'mymaven'
    }
    
    stages{
        stage('Checkout code'){
            agent any
            steps{
                git 'https://github.com/Paras116255/health-care-project002-devops.git'
            }
        }
        stage('Package code'){
            agent any
            steps{
                sh 'mvn package'
            }
        }
        stage('Build Image'){
            agent any
            steps{
                sh 'docker build -t myimage:project2 .'
            }
        }
        stage('push the image to docker hub'){
            agent any
            steps{
                withCredentials([string(credentialsId: 'DOCKER_HUB_PASWD', variable: 'DOCKER_HUB_PASWD')]) {
                sh 'docker login -u paras1112 -p ${DOCKER_HUB_PASWD}'
                }
                sh 'docker tag myimage:project2 paras1112/myimage:project2'
                sh 'docker push paras1112/myimage:project2'
            }
        }
         stage('Deploy on Test server')
        {
            agent {
                label 'test_server'
                
            }
            steps{
                sh 'docker run -d -P paras1112/myimage:project2 '
            }
        }

    }
    
}


Create a Kubernetes cluster of your choice
 
 


# vim deployment1.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mydeploy
spec:
  replicas: 3
  selector:
    matchLabels:
      type: webserver
  template:
    metadata:
      labels:
        type: webserver
    spec:
      containers:
        - name: c1
          image: paras1112/myimage:project2
# vim service.yml
apiVersion: v1
kind: Service
metadata:
  name: mysvc1
spec:
  type: NodePort
  ports:
    - targetPort: 8081
      port: 8081
  selector:
    type: webserver


 # kubectl create -f deployment1.yml
 # kubectl get all
 # kubectl get pods --show-labels
 # kubectl create -f service.yml
 # kubectl get endpoints
