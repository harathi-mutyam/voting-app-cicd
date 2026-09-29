# Production-Grade AWS EKS Deployment(Terraform) Voting App 

A simple distributed application running across multiple Docker containers.

**This repository contains the complete infrastructure and application setup for the Example Voting Application. However, in this documentation, only the Kubernetes deployment procedure for deploying the application on Amazon EKS is explained.**

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.

## Prerequisites Setup for This Repository 
```shell
- AWS Load Balancer Controller (ALB)

- Install AWS CLI using Chocolatey  aws --version

- Install Terraform using Chocolatey  terraform -version
```
# Step 1: eks-voting-app – Terraform Deployment Guide

## 1. Clone the Repository 
### Locally using terminal
  git clone https://github.com/harathi-mutyam/kubernetes-projects.git

or 

### Using VS Code

**In vs code-->click on terminal-->new terminal-->select git bash-->Navigate to your desired local directory --> run git clone command**


```shell

git clone https://github.com/harathi-mutyam/kubernetes-projects.git

```
after cloning finished--->cick on file-->open Folder-->select your cloned repository(project folder)

```shell
ls
cd eks-voting-app

ls
cd terraform
ls
cd EKS
```
Make sure this folder contains: main.tf ,eks.tf,vpc.tf,provider.tf,dev.tfvars

## 2. Configure AWS CLI (mandatory):** for that go to browser create a IAM user (eks-user) in aws console for this project save access key and secret keys locally
```shell
aws configure

Set:

AWS Access Key : give your access key
Secret Key  : give your secret key
Region → us-east-1
json
```
## 3. Create S3 backend bucket (if not already created)

Through aws console or through command
```shell
aws s3 mb s3://your-terraform-state-bucket --region us-east-1
```

in **backend.tf** change the bucket name 

bucket       = "backend-bucket-final-6526"   #create s3 bucket through aws console .and replace bucket name here

Update the region based on your AWS region. in backend block 

### Make changes in dev.tfvars 

Change Ami id of Ubuntu Server Based on region

ami_id        = "ami-091138d0f0d41ff90"  #replace with your ubuntu ami id based on region

Instance type based on your project size

instance_type = "c7i-flex.large"  #change this to t3.small

aws_region                = "us-east-1"   #change region

instance_types   = ["c7i-flex.large"]  # t3.small

if reguired change 

kubernetes_version        = "1.34" also

## 4. Create EC2 Key Pair (VERY IMPORTANT) manually through aws console
## 5. Initialize Terraform
```shell
terraform init
```  
This will: download AWS provider ,initialize backend (S3) ,prepare modules

## 6. Validate configuration**
```shell
terraform validate
```
## 7. Plan infrastructure**
```shell
terraform plan -var-file="dev.tfvars"    #because all global configuration settings are defined in dev.tfvars. If you run only terraform plan, it will prompt you to enter values manually.”
```
We use dev.tfvars with terraform plan so Terraform already knows all values. If we don’t use it, Terraform will stop and ask us to enter them one by one.

## 8. Apply infrastructure**
```shell
terraform apply -var-file="dev.tfvars"

Type:yes
```
After resource creation completes, verify in the AWS Console that the bastion server, cluster, and VPCs ,IAM Roles are available. The process usually takes 10–15 minutes. Then copy the bastion server’s public IP address and launch a new Git Bash.

# Step 2: Post Deployment (Bastion Access & Kubernetes Setup)

### 1. Access Bastion Host
**Copy Bastion public IP from AWS Console**
**Open new Git Bash terminal**
**Connect using SSH:**

```shell
ssh -i Downloads/ec2_keypair.pem ubuntu@bastion_public_ip
```

### 2. Configure AWS
```shell
aws configure    #enter here access keys and secret keys ,region and json

AWS Access Key : give your saved access key

Secret Key  : give your saved secret key

Region → us-east-1

json
```
### 3. Configure Kubernetes access
```shell
aws eks update-kubeconfig --region us-east-1 --name dev-eks-demo
```
**Verify:**
```shell
kubectl get nodes
```
### 4. Install Helm (inside bastion)
```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
## 5. Install AWS Load Balancer Controller
```shell
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install:

```shell
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=(dev-eks-demo) \
  --set region=us-east-1 \
  --set vpcId=(vpc-id) \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=(IAM-role-arn)
```

replace with your clustername ,vpc id and iam role of loadbalancer(search in aws console iam-->roles-->search  AWSLoadBalancerControllerRole   copy the arn ) arn and remove () also

## 6. Clone your repository on the Bastion Server for Deploying the Voting Application
Recommended Deployment Order

To ensure all application components start correctly and dependencies are available, deploy the Kubernetes resources in the following order.

```shell
git clone https://github.com/harathi-mutyam/kubernetes-projects.git
ls
cd kubernetes-projects
ls
cd eks-voting-app
ls
cd k8s-specifications
ls
```
### Step 1:  Create Namespace
Create a dedicated namespace for the voting application:
```shell
kubectl create namespace voting-app

```
```shell
 kubectl get ns

ls
```
### Step 2: open  alb-ingress.yaml file replace host: vote.yourdomain.com
```shell
vim alb-ingress.yaml

```
save the file :wq!

**Note: create a Domain in Godaddy later** 


### Step 3: Deploy all Manifests in the following order

**Deploy PostgreSQL Database**

Deploy the PostgreSQL database first because the worker service depends on it.

```shell
 
kubectl apply -f db-deployment.yaml -n voting-app

kubectl apply -f db-service.yaml -n voting-app

```
**Deploy Redis**

Redis acts as the message queue between the voting application and the worker service.

```shell
kubectl apply -f redis-deployment.yaml -n voting-app

kubectl apply -f redisservice.yaml -n voting-app
```

```shell
 
 kubectl apply -f worker-deployment.yaml -n voting-app
```

**Deploy Voting Application**

Deploy the frontend voting application.

```shell
 kubectl apply -f vote-deployment.yaml -n voting-app

 kubectl apply -f vote-service.yaml -n voting-app
```

**Deploy Result Application**

Deploy the result application used to display voting statistics.

```shell
 kubectl apply -f result-deployment.yaml -n voting-app

   kubectl apply -f result-service.yaml -n voting-app

```
**Deploy Worker Service**
```shell
The worker transfers votes from Redis to PostgreSQL.

kubectl apply -f worker-deployment.yaml -n voting-app
```
**Deploy ALB Ingress**

Finally, expose the applications externally using AWS Application Load Balancer (ALB) Ingress.

```shell
 kubectl apply -f alb-ingress.yaml
```
## Step 4.Verify Deployment

**Check all pods:**
```shell

kubectl get pods -n voting-app
```

**Check services:**

```shell
kubectl get svc -n voting-app
```
**Check ingress:**
```shell

kubectl get ingress -n voting-app

```
**You will see:**

**ALB DNS name → k8s-default-xxxx.elb.amazonaws.com**

## Step 5: Connect Domain → ALB using Route 53

## You can use either either procedure 1((Recommended for AWS-native setup) or procedure 2(GoDaddy-managed DNS).

# PROCEDURE 1 (Recommended – Route 53 Managed DNS)

### Route53 Setup

**Create Route53 Hosted Zone**

Go to AWS Console → Route53 --> Click Hosted Zones --> Click on Create Hosted Zone --->

Domain: ehmutyam.xyz Type: choose Public Hosted Zone Click Create

You will see 2 important things: NS (Name Servers) → 4 values SOA record

Example: 4 values of ns servers

ns-123.awsdns-45.org
ns-456.awsdns-90.com
ns-789.awsdns-12.net
ns-222.awsdns-34.co.uk
copy ns records


### Update GoDaddy Nameservers

go to GoDaddy in browser:

Login Go to: My Products → ehmutyam.xyz → DNS --> Find Nameservers Tab --> Click Change

Select:

Custom Nameservers : select “I’ll use my own nameservers”

Paste the 4 Route53 NS records without end . (remove last . from NSServers values)

for example:

ns-xxx.awsdns-xx.org
ns-xxx.awsdns-xx.com
ns-xxx.awsdns-xx.net
ns-xxx.awsdns-xx.co.uk
Save
**Configure DNS Records for the ALB**

Later add A or CNAME records in R53 under domain name

AWS Console → Route53 → Hosted Zones → Your Domain

**Create records for each application:**

```shell
RecordType	   Name	    Target
A (Alias)	     vote	    ALB DNS Name
A (Alias)	      result	ALB DNS Name
```
Point both records to:

k8s-votingapp-xxxxxxxx.us-east-1.elb.amazonaws.com
Alias → Application Load Balancer
```shell
steps in detail:

REcord Name: **vote ** Record Type: A record

Enable: Alias = YES

Route Traffic to : select Alias to Application and Classic Load Balancer

select region of your ALB : US East(N. Virginia) 

select your ALB DNS here (for example: k8s-votingapp-xxxxxxxx.us-east-1.elb.amazonaws.com)


```
**Check it in browser :** with host names To get output it will take some time 

#### DNS propagation usually takes:

#### 5–30 minutes
#### Sometimes up to 1 hour

http://vote.ehmutyam.xyz 

http://result.ehmutyam.xyz

# PROCEDURE 2 GoDaddy CNAME Method (Without Route 53 Delegation)

## Step 1: Use GoDaddy Default Nameservers

If you previously added Route 53 NS records in GoDaddy, remove them and switch back to GoDaddy default nameservers.

This step is optional if your domain is already using GoDaddy DNS management.

You do NOT need Route 53 hosted zone delegation for this method.

## Step 2: Create CNAME records in godaddy.com for http purpose 

Open:

GoDaddy → My Products → DNS Management

**Create CNAME records:**
```shell
Type	   Name	   Value
CNAME	   vote	    ALB DNS Name
CNAME	    result	 ALB DNS Name
```
```shell
Example:

Type	Name	Value
CNAME	vote	k8s-votingapp-xxxx.us-east-1.elb.amazonaws.com
CNAME	result	k8s-votingapp-xxxx.us-east-1.elb.amazonaws.com
```

## Step 6: Check DNS propagation:
ehmutyam.xyz in my domain name replace with your domain name

```shell
nslookup vote.ehmutyam.xyz
nslookup vote.ehmutyam.xyz 8.8.8.8
dig vote.ehmutyam.xyz
curl -H "Host: vote.ehmutyam.xyz" http://k8s-votingapp-015023cb5f-1200792225.us-east-1.elb.amazonaws.com
curl -H "Host: result.ehmutyam.xyz" http://k8s-votingapp-015023cb5f-1200792225.us-east-1.elb.amazonaws.com
```

# For Debugging purpose use the below commands
```shell
#Verify whether votes are being processed correctly by the worker application.

kubectl logs deploy/worker -n voting-app

# Verify PostgreSQL Database Connectivity
# List PostgreSQL databases:

kubectl exec -it deploy/db -n voting-app -- psql -U postgres -c "\l"

#Login into PostgreSQL shell:

kubectl exec -it deploy/db -n voting-app -- psql -U postgres

#Follow live logs:

kubectl logs deploy/worker -n voting-app -f

output will be Processing vote for 'a' by 'f5dff6e94e55cd0'

press ctrl + c

#Verify Votes Table Data
#Display all votes stored in PostgreSQL:

kubectl exec -it deploy/db -n voting-app -- psql -U postgres -d postgres -c "select * from votes;"

#Count total votes:

kubectl exec -it deploy/db -n voting-app -- psql -U postgres -c "select count(*) from votes;"

kubectl exec -it deploy/db -n voting-app -- psql -U postgres -d postgres -c "select * from votes;"
#T est Voting and Result Applications

 curl result.ehmutyam.xyz
 
 curl vote.ehmutyam.xyz

#Test ALB Host-Based Routing .Verify ALB ingress host routing manually:
 
curl -H "Host: vote.ehmutyam.xyz" http://k8s-votingapp-015023cb5f-1200792225.us-east-1.elb.amazonaws.com
```

## Step 7: Access Application in Browser

Once DNS propagation completes:

http://vote.yourdomain.com

http://result.yourdomain.com

## Step 8: Process of Deletion
**delete load balancer from baston server** gitbash first
```shell
kubectl delete -f alb-ingress.yaml
```
run this command in vs code gitbash

**Destroy Infrastructure (Cleanup)**
```shell
terraform destroy -var-file="dev.tfvars"
```
**Step 3: Manual Cleanup Steps**
After destroying the infrastructure using Terraform, manually remove the remaining AWS and DNS resources to avoid unnecessary charges and unused configurations.

Delete Route53 Hosted Zone

Remove Route53 Nameservers from Domain Registrar (GoDaddy).If you followed Approach 1 (Route53 Nameserver Integration):

Delete DNS Records from GoDaddy (If Approach 2 Was Used)

Delete S3 Bucket Used for Terraform State: First Empty the bucket later delete it

