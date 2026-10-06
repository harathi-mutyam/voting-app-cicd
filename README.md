# Voting App — Final Presentation Runbook
This is the cleaned-up version for your next project presentation/trial.
The starting point is your existing GitHub repository:
voting-app-cicd GitHub repository 
The repository already contains the application source code, CI/CD workflow, Terraform configuration, Kubernetes manifests, and SonarQube configuration.
Your Docker Hub repositories already exist:
•	harathi2026/voting-vote
•	harathi2026/voting-result
•	harathi2026/voting-worker
Therefore, do not manually build the three Docker images before starting the presentation. GitHub Actions will build them and push them to Docker Hub.

---

## 1. Final Architecture
                       Developer
                           |
                           v
                    GitHub Repository
                   voting-app-cicd
                           |
                           v
                    GitHub Actions
                           |
                 Self-Hosted EC2 Runner
                           |

```text
          +----------------+----------------+
          |                |                |
       Python CI        Node.js CI       Docker
          |                |              Build
          +----------------+----------------+
                           |
                       Gitleaks
                           |
                       SonarQube
                           |
                    Docker Build & Push
                           |
                           v
                       Docker Hub
                           |
                           v
                         EKS
                           |
                        Argo CD
                           |
                           v
                   Voting Application
              +------------+------------+
              |            |            |
             Vote        Result       Worker
              |            |            |
            Redis      PostgreSQL      Redis
                           |
                           v
                  Prometheus + Grafana
```

---

## 2. Important Difference From Your First Trial
For your first trial, you manually tested Docker images:

```bash
docker build -t voting-vote:local ./vote
docker build -t voting-result:local ./result
docker build -t voting-worker:local ./worker
```
For the presentation, these steps are unnecessary.
The CI/CD pipeline itself performs:	
GitHub

```text
   ↓
```
GitHub Actions

```text
   ↓
```
Docker Build

```text
   ↓
```
Docker Hub

```text
   ↓
```
EKS
So you only need to make sure Docker is installed and working on the self-hosted GitHub Actions runner.

---

## 3. Prerequisites
Before beginning the presentation, have:
•	AWS account
•	GitHub account
•	Docker Hub account
•	Existing GitHub repository
•	Existing Docker Hub repositories
•	AWS key pair
•	IAM credentials with sufficient permissions for the Terraform/EKS setup
•	Your GitHub repository secrets/variables available
Repository:
https://github.com/harathi-mutyam/voting-app-cicd.git
Docker Hub:
harathi2026/voting-vote
harathi2026/voting-result
harathi2026/voting-worker

---

## 4. Clone the Existing Repository
On your local computer:

```bash
cd /d/harathi
```
Clone the existing repository:

```bash
git clone https://github.com/harathi-mutyam/voting-app-cicd.git
```
Enter the project:

```bash
cd voting-app-cicd
```
Check:

```bash
pwd
ls
```
You should see the project files.
Check the workflow:

```bash
cat .github/workflows/ci.yml
```
Check Terraform:

```bash
ls terraform
```
Check Kubernetes:

```bash
ls k8s-specifications
```
Check SonarQube configuration:

```bash
cat sonar-project.properties
```

---

## 5. Do NOT Remove .git
In your first trial you used:

```bash
rm -rf .git
git init
```
That was necessary because you were moving existing project code into a new repository.
Do not do that now.
You are cloning the correct repository directly.
Keep the existing Git configuration.
Check:

```bash
git remote -v
```
Expected:
origin  https://github.com/harathi-mutyam/voting-app-cicd.git

---

## 6. Create GitHub Actions Runner EC2
Create an EC2 instance.
Configuration
Name: voting-app-cicd-runner
OS: Ubuntu
Instance type: c7i-flex.large
Key pair: github-key
Security Group: githubaction_sg
Storage: 35 GB

---

## 7. Security Group
Create:
githubaction_sg
Recommended rules for your learning setup:
Type	Port	Source
SSH	22	My IP
HTTP	80	Anywhere
HTTPS	443	Anywhere
Custom TCP	9000	My IP
Custom TCP	3000–11000	Anywhere IPv4
For a real production environment, avoid opening broad port ranges to the internet unless specifically required.

---

## 8. Connect to Runner EC2
From Git Bash:

```bash
ssh -i Downloads/github-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```
Set hostname:

```bash
sudo hostnamectl set-hostname voting-app-cicd-runner
```
/bin/bash
Update:

```bash
sudo apt update
sudo apt upgrade -y
```
Check OS:

```bash
cat /etc/os-release
```

---

## 9. Install Git
Check:

```bash
git --version
```
If missing:

```bash
sudo apt install git -y
```
Configure Git:

```bash
git config --global user.name "harathi-mutyam"
git config --global user.email "YOUR_EMAIL"
```
Verify:

```bash
git config --global --list
```

---

## 10. Clone Repository on Runner optional with github authentication

```bash
cd ~
git clone https://github.com/harathi-mutyam/voting-app-cicd.git
```
Enter:

```bash
cd ~/voting-app-cicd
```
Check:

```bash
pwd
ls
```
Expected:
/home/ubuntu/voting-app-cicd
Check files:

```bash
find . -maxdepth 2 -type f
```


## 10. GitHub Personal Access Token (PAT) — First-Time Git Authentication
Create the GitHub Personal Access Token during the initial Git setup, before cloning the private repository or performing the first git push.

### Step 1: Generate GitHub Personal Access Token
Go to:
GitHub --> Profile --> Settings --> Developer settings --> Personal access tokens --> Fine-grained tokens --> Generate new token
Configure:
Token name: voting-runner

Expiration: Choose the required expiration period

Repository access: Only select repositories

Select Repository: harathi-mutyam/voting-app-cicd

Repository permissions: Contents --> Read and write

Actions --> Read and Write
Optional Noes: If you are going to modify or push changes to files under:
.github/workflows/
also give the token the workflow-related permission required by the current GitHub interface.
Click: Generate token
⚠️ Copy the token immediately and keep it secure.
Do NOT put the actual token in this document, GitHub repository, YAML file, or chat.

### Step 2: Clone Repository Using GitHub Authentication

```bash
cd ~
git clone https://github.com/harathi-mutyam/voting-app-cicd.git
```
If Git asks:
Username: enter:  harathi-mutyam
When Git asks:  Password:  paste the GitHub PAT.
The PAT is used instead of your normal GitHub password.
Then:

```bash
cd ~/voting-app-cicd
```

### Step 3: Verify Repository

```bash
pwd
ls
git remote -v
```
Expected:
origin  https://github.com/harathi-mutyam/voting-app-cicd.git

### Step 4: First Git Push
When you later make changes to the repository:

```bash
git status
git add .
git commit -m "Update voting app CI"
git push origin main
```
If Git asks for authentication again, use:
Username: harathi-mutyam

Password: < paste GitHub PAT>
After the push succeeds:
GitHub --> voting-app-cicd --> Actions
The ci.yml workflow will automatically start because it is configured for:

```yaml
on:
  push:
    branches:
      - main
```
Important
The PAT does not need to be generated every time.
Generate it once during the initial Git setup. During the presentation, if you are creating the environment from scratch, you can generate a fresh PAT at this point.


---

## 11. Install Python
Check:

```bash
python3 --version
pip3 --version
```
If pip is missing:

```bash
sudo apt install python3-pip -y
```
Check:

```bash
pip3 --version
```
Check application requirements:

```bash
cat vote/requirements.txt
```
You can test them:

```bash
cd ~/voting-app-cicd/vote
pip3 install -r requirements.txt --break-system-packages
python3 -m compileall .
```
Return:

```bash
cd ~/voting-app-cicd   or cd ..
```

---

## 12. Install Node.js
Check:

```bash
node --version
npm --version
```
If missing:

```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```
Verify:

```bash
node --version
npm --version
```
Check application:

```bash
cat result/package.json
ls -lh result/package-lock.json
```
Install dependencies:

```bash
cd ~/voting-app-cicd/result
npm ci
```
Verify:

```bash
ls -ld node_modules
npm list --depth=0
```
Return:

```bash
cd ~/voting-app-cicd
```

---

## 13. Install Docker
Check:

```bash
docker --version
```
If Docker is not installed:

```bash
sudo apt remove -y docker.io docker-doc docker-compose podman-docker containerd runc
```


```bash
sudo apt update
```


```bash
sudo apt install -y ca-certificates curl
```


```bash
sudo install -m 0755 -d /etc/apt/keyrings
```


```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```


```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```


```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo ${UBUNTU_CODENAME:-$VERSION_CODENAME}) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


```bash
sudo apt update
```


```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Check:

```bash
docker --version
docker compose version
```
Allow the Ubuntu user to use Docker:

```bash
sudo usermod -aG docker $USER
newgrp docker
```
Test:

```bash
docker run hello-world
```
You should see:
Hello from Docker!
Optional no need it Important
Do not manually build the voting application images here.
You do not need:

```bash
docker build -t voting-vote:local ./vote
docker build -t voting-result:local ./result
docker build -t voting-worker:local ./worker
```
GitHub Actions will perform the Docker builds later.

---

## 14. Docker Hub
Your repositories should already exist:
harathi2026/voting-vote
harathi2026/voting-result
harathi2026/voting-worker
They should be public for this learning setup.
If you are not found it create using below procedure
Use your dockerhub user details
Log in to Docker Hub:
# Create Docker Hub repositories
Create these three repositories:

## 1.	voting-vote

## 2.	voting-result

## 3.	voting-worker
output will be : harathi2026/voting-vote ,  harathi2026/voting-result , harathi2026/voting-worker
Repository visibility must be public for all 3 repositories
That means later EKS can pull them without requiring a Kubernetes Docker Hub secret.
# Create Docker Hub Access Token
In Docker Hub:
Account Settings --> Personal access tokens --> Generate access token
Create a token specifically for this project.
For example, name it:  voting-github-actions
Access token description:  eks-voting-github-actions
Expiration: 30 days or your choice
Access permissions,: select Read & Write  (Give it permission to push images.)
-->Then click on generate button 
Docker Hub will show the token only when it's created, so copy it somewhere temporarily.
Create/use a Docker Hub access token with permission to push images.
Do not put the token inside your GitHub repository.

---

## 15. GitHub Repository Variables and Secrets
Open:
GitHub --> voting-app-cicd --> Settings --> Secrets and variables --> Actions
Select Repository variable
Create:
Name: DOCKERHUB_USERNAME

Value: harathi2026
Repository secret
Create:
Name: DOCKERHUB_TOKEN

Value: YOUR_DOCKER_HUB_ACCESS_TOKEN
You should have: Variables: DOCKERHUB_USERNAME   --> Secrets: DOCKERHUB_TOKEN

---

## 16. Install GitHub Self-Hosted Runner
Open:
GitHub
--> voting-app-cicd --> Settings--> Actions--> Runners--> New self-hosted runner
Select:
Linux
x64
GitHub will display the current runner installation commands.
On EC2:

```bash
cd ~
mkdir actions-runner
cd actions-runner
```
Run the download command provided by GitHub.
It will look similar to:

```bash
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/...
```
Extract:

```bash
tar xzf ./actions-runner.tar.gz
```
Configure:
./config.sh --url https://github.com/harathi-mutyam/voting-app-cicd --token YOUR_GITHUB_RUNNER_TOKEN
Press press enter key only don’t type voting-app-cicd
When prompted:
Runner name:
Press Enter or provide: voting-app-cicd-runner or press enter key
Labels: self-hosted
Work folder: press Enter key
Start:
./run.sh
You should see:
Connected to GitHub
Listening for Jobs
Keep this terminal running during the presentation.

---

## 17. Verify Runner
Go to:
GitHub --> voting-app-cicd --> Settings --> Actions --> Runners
You should see something similar to:
voting-app-cicd-runner
Idle
That confirms the EC2 machine is connected.

---







## 18. Your Final ci.yml
Your repository should contain:
.github/

```text
└── workflows/
    └── ci.yml
```
The final workflow should be:

```yaml
name: Voting App CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:

  vote-ci:
    name: Vote - Python CI
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check Python
        run: python3 --version

      - name: Install vote dependencies
        working-directory: vote
        run: |
          python3 -m pip install -r requirements.txt --break-system-packages

      - name: Python syntax check
        working-directory: vote
        run: |
          python3 -m compileall .


  result-ci:
    name: Result - Node.js CI
    needs: vote-ci
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check Node.js
        run: node --version

      - name: Check npm
        run: npm --version

      - name: Install dependencies
        working-directory: result
        run: npm ci


  worker-ci:
    name: Worker - Docker Build
    needs: result-ci
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check Docker
        run: docker --version

      - name: Build worker image
        run: |
          docker build -t voting-worker:ci ./worker


  gitleaks:
    name: Gitleaks Secret Scan
    needs: worker-ci
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}


  sonarqube:
    name: SonarQube Code Analysis
    needs: gitleaks
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check Java
        run: |
          java -version

      - name: Check SonarScanner
        run: |
          which sonar-scanner
          sonar-scanner --version

      - name: Test SonarQube Authentication
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        run: |
          echo "Testing SonarQube authentication..."

          if [ -z "$SONAR_TOKEN" ]; then
            echo "ERROR: SONAR_TOKEN is empty"
            exit 1
          fi

          if [ -z "$SONAR_HOST_URL" ]; then
            echo "ERROR: SONAR_HOST_URL is empty"
            exit 1
          fi

          RESPONSE=$(curl -sS \
            -u "$SONAR_TOKEN:" \
            "$SONAR_HOST_URL/api/authentication/validate")

          echo "Authentication response:"
          echo "$RESPONSE"

          if echo "$RESPONSE" | grep -q '"valid":true'; then
            echo "SUCCESS: SonarQube authentication successful"
          else
            echo "ERROR: SonarQube authentication failed"
            exit 1
          fi

      - name: Run SonarQube Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
        run: |
          echo "Starting SonarQube analysis..."

          sonar-scanner \
            -Dsonar.host.url="$SONAR_HOST_URL" \
            -Dsonar.login="$SONAR_TOKEN"

          echo "SonarQube analysis completed successfully"


  docker-build-push:
    name: Build and Push Docker Images
    needs: sonarqube
    runs-on: self-hosted

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Check Docker
        run: docker --version

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Vote image
        run: |
          docker build \
            -t ${{ vars.DOCKERHUB_USERNAME }}/voting-vote:latest \
            ./vote

      - name: Build Result image
        run: |
          docker build \
            -t ${{ vars.DOCKERHUB_USERNAME }}/voting-result:latest \
            ./result

      - name: Build Worker image
        run: |
          docker build \
            -t ${{ vars.DOCKERHUB_USERNAME }}/voting-worker:latest \
            ./worker

      - name: Push Vote image
        run: |
          docker push \
            ${{ vars.DOCKERHUB_USERNAME }}/voting-vote:latest

      - name: Push Result image
        run: |
          docker push \
            ${{ vars.DOCKERHUB_USERNAME }}/voting-result:latest

      - name: Push Worker image
        run: |
          docker push \
            ${{ vars.DOCKERHUB_USERNAME }}/voting-worker:latest
```
The needs sequence means:
vote-ci  --> result-ci  -->worker-ci -->gitleaks -->sonarqube -->docker-build-push
This gives you a clear pipeline to demonstrate.

---

## 19. SonarQube EC2
Create another EC2:
Name: sonarqube-server
OS: Ubuntu
Instance type: c7i-flex.large
Key pair: github-key
Security Group: githubaction_sg
Storage: 40 GB
Connect:

```bash
ssh -i Downloads/github-key.pem ubuntu@YOUR_SONARQUBE_PUBLIC_IP
```
Set hostname:

```bash
sudo hostnamectl set-hostname sonarqube
```
/bin/bash
Update:

```bash
sudo apt update
sudo apt upgrade -y
```

---

## 20. Install Docker on SonarQube Server

```bash
sudo apt install -y ca-certificates curl
```


```bash
sudo install -m 0755 -d /etc/apt/keyrings
```


```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```


```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```


```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo ${UBUNTU_CODENAME:-$VERSION_CODENAME}) stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```


```bash
sudo apt update
```


```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Verify:

```bash
docker --version
docker compose version
```
Enable Docker:

```bash
sudo usermod -aG docker $USER
newgrp docker
```
Test:

```bash
docker run hello-world
```

---

## 21. Run SonarQube

```bash
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  sonarqube:lts-community
```
Check:

```bash
docker ps
```
Logs:

```bash
docker logs sonarqube
```
Wait for SonarQube to start.
Open: in Browser
http://SONARQUBE_PUBLIC_IP:9000
Default:
Username: admin
Password: admin
Change the password will open.change your password

---
Optional step

## 22. Create SonarQube Project
In SonarQube:
Projects
--> Create Project
--> Create a project manually
Use:
Project display name:
voting-app-cicd

Project key:
voting-app-cicd

---

## 23. Generate SonarQube Token
In Browser Go to:
Administration --> Security --> Users
Generate a token.  --> click on this symbol   --> Enter Name: A or anything  -->select expires 30 days or based on your requiremnt
Save it temporarily.
Do not put the token into GitHub source code.

## 24. SonarQube GitHub Secret
 	GitHub:
Settings  --> Secrets and variables  --> Actions 

Create secret: 
Name: SONAR_TOKEN
Value: YOUR_SONARQUBE_TOKEN
Create variable:
Name: SONAR_HOST_URL
Value:  http://YOUR_SONARQUBE_PUBLIC_IP:9000

---

## 25. sonar-project.properties
The repository should contain:
sonar-project.properties
Contents:
sonar.projectKey=voting-app-cicd
sonar.projectName=voting-app-cicd
sonar.sources=.
sonar.exclusions=**/node_modules/**,**/__pycache__/**,**/target/**,**/.git/**,**/bin/**,**/obj/**

---

## 26. Install Java and SonarScanner on Runner
Go back to the GitHub Actions runner EC2.

```bash
ssh -i Downloads/github-key.pem ubuntu@YOUR_RUNNER_PUBLIC_IP
```


```bash
sudo hostnamectl set-hostname runnerinstallation
```
/bin/bash

Install required packages:

```bash
sudo apt update
sudo apt install -y unzip curl
```
Check Java:

```bash
java -version
```
If Java is missing, install a supported JDK.

```bash
sudo apt install -y openjdk-17-jdk
java -version
curl --–rsion
docker --–rsion
npm --–rsion
```
Download SonarScanner using the current version appropriate for your environment.

```bash
sudo npm install -g @sonar/scan
which sonar-scanner
```
optinal commands :Download SonarScanner using the current version appropriate for your environment.
Example:

```bash
rm -f sonar-scanner.zip
```


```bash
curl -fL -o sonar-scanner.zip \
https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-7.1.0.5888-linux-x64.zip
```
Test:

```bash
unzip -t sonar-scanner.zip
```
Install/configure it so that:

```bash
which sonar-scanner
```

sonar-scanner --version
work successfully.
The important requirement is that the runner can execute:
sonar-scanner --version

---

## 27. Test SonarQube Connectivity
From the runner:

```bash
curl -I http://YOUR_SONARQUBE_PUBLIC_IP:9000
```
You should receive an HTTP response. HTTP/1.1 200 

---

## 28. Run GitHub Actions
Push the workflow if you made any changes optional comand

```bash
cd ~/voting-app-cicd
```


```bash
git status
git pull origin main
```

Make a small CI-triggering change
For example, open the workflow: vim .github/workflows/ci.yml
Make the required change or correction and save:
Esc
:wq
Enter
# Check the change

```bash
git diff
```
# Add the workflow

```bash
git add .github/workflows/ci.yml
```
#  Commit

```bash
git commit -m “Update voting app CI”
```
# Push to GitHub

```bash
git push origin main
```
If Git asks for authentication: 
Username:  harathi-mutyam

Password: GitHub Personal Access Token (PAT)

Then open in browser:  GitHub  --> voting-app-cicd--> Actions  --> you should automatically see
You should see:
Voting App CI
The jobs execute in sequence:
Vote - Python CI  --> Result - Node.js CI --> Worker - Docker Build  --> Gitleaks Secret Scan   -->  SonarQube Code Analysis  --> Build and Push Docker Images

---

## 29. Verify Docker Hub
After the final job succeeds, open Docker Hub.
Check: harathi2026/voting-vote  , harathi2026/voting-result , harathi2026/voting-worker
You should see: latest 
The important presentation point is:
Docker images are automatically built by GitHub Actions and pushed to Docker Hub. No manual local image build is required.

---

## 30. Create EKS Server EC2
Create another EC2 instance:
Name: Server
OS: Ubuntu
Instance type: c7i-flex.large
Key pair: github-key
Security Group: githubaction_sg
Storage: 25 GB
Connect:

```bash
ssh -i Downloads/github-key.pem ubuntu@YOUR_SERVER_PUBLIC_IP
```
Set hostname:

```bash
sudo hostnamectl set-hostname server
```
/bin/bash
Update:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install unzip curl -y
```

---

## 31. Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
-o "awscliv2.zip"
```


```bash
unzip awscliv2.zip
```


```bash
sudo ./aws/install
```
Verify:

```bash
aws --version
```

Create Access key and Secret Keys for Iam User in Aws Console:
in browser --> aws console -->Create an access keys and secret keys for IAM user
save the access key and secret keys
Configure:

```bash
aws configure
```
Example:
Default region:  eu-north-1

Default output: json
Verify:

```bash
aws sts get-caller-identity
```

---

## 32. Install Terraform

```bash
sudo apt-get update
```


```bash
sudo apt-get install -y \
gnupg \
software-properties-common \
curl \
wget
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update
sudo apt-get install -y terraform
```
Verify:

```bash
terraform version
```

---

## 33. Clone Repository on Server

```bash
cd ~
```


```bash
git clone https://github.com/harathi-mutyam/voting-app-cicd.git
```
Enter:

```bash
cd ~/voting-app-cicd
```
Check:

```bash
find . -name "*.tf"
```
Go to Terraform:

```bash
cd terraform
```

---

## 34. Create EKS With Terraform
Format:

```bash
terraform fmt
```
Initialize:

```bash
terraform init
```
Validate:

```bash
terraform validate
```
Plan:

```bash
terraform plan
```
Apply:

```bash
terraform apply --auto-approve
```
If your Terraform configuration requires importing pre-existing IAM roles, use the appropriate import commands only when Terraform reports that situation.
For example:

```bash
terraform import aws_iam_role.eks_cluster_role eks-cluster-role
terraform import aws_iam_role.eks_node_group_role eks-node-group-role
```
Then:

```bash
terraform plan
terraform apply --auto-approve
```

---

## 35. Verify EKS

```bash
aws eks list-clusters --region eu-north-1
```
Check cluster:

```bash
aws eks describe-cluster \
  --region eu-north-1 \
  --name eks-cluster \
  --query "cluster.status"
```

---

## 36. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
Verify:

```bash
kubectl version --client
```

---

## 37. Configure kubectl for EKS

```bash
aws eks update-kubeconfig \
  --region eu-north-1 \
  --name eks-cluster
```
Verify:

```bash
kubectl get nodes
```
Then:

```bash
kubectl get pods -A
kubectl cluster-info
```

---

## 38. Kubernetes Image Configuration
Go to the project:

```bash
cd ~/voting-app-cicd
```
Check:

```bash
ls k8s-specifications
```
The deployment files should use your Docker Hub images.
Vote  image: harathi2026/voting-vote:latest
Result   image: harathi2026/voting-result:latest
Worker  image: harathi2026/voting-worker:latest
Do not use:  dockersamples/examplevotingapp_...
for your final presentation configuration.

---

## 39. Kubernetes Services
For the presentation, your Vote and Result services should be exposed externally.
Check:

```bash
cat k8s-specifications/vote-service.yaml
cat k8s-specifications/result-service.yaml
```
Use:  type: LoadBalancer   
for Vote and Result.
Make sure the target port matches the application container configuration.

---

## 40. Deploy Kubernetes Application
From:

```bash
cd ~/voting-app-cicd
```
Run:

```bash
kubectl apply -f k8s-specifications/
```
Check:

```bash
kubectl get deployments -n voting-app
```
Pods:

```bash
kubectl get pods -n voting-app
```
Services:

```bash
kubectl get services -n voting-app
```
You should eventually see LoadBalancer hostnames for Vote and Result.

---

## 41. Troubleshooting Kubernetes
If a pod is not running:

```bash
kubectl describe pod POD_NAME -n voting-app
```
Check logs:

```bash
kubectl logs POD_NAME -n voting-app
```
Check everything:

```bash
kubectl get pods -n voting-app -o wide
kubectl get endpoints -n voting-app
kubectl get deployments -n voting-app
kubectl get services -n voting-app
```
Check service connectivity:

```bash
curl -v http://YOUR_LOADBALANCER_HOSTNAME
```

---

## 42. Test Voting Application
Get services:

```bash
kubectl get svc -n voting-app
```
Copy the Vote LoadBalancer hostname.
Open:  http://VOTE-LOADBALANCER
Then open the Result LoadBalancer:
http://RESULT-LOADBALANCER
This demonstrates that the application is running on EKS.
Wait for 2 minutes to check it in browser

---

## 43. Install Argo CD
Create namespace:

```bash
kubectl create namespace argocd
```
Install:

```bash
kubectl apply -n argocd \
--server-side \
--force-conflicts \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Check:

```bash
kubectl get pods -n argocd
```
Wait until the pods are running.

---

## 44. Expose Argo CD
Check:

```bash
kubectl get svc -n argocd
```
Change: argocd-server   to LoadBalancer:

```bash
kubectl edit svc argocd-server -n argocd
```
Change:
type: ClusterIP   to:  type: LoadBalancer
Save:
Esc  , :wq   ,  Enter
Check:

```bash
kubectl get svc argocd-server -n argocd
```
Wait for the AWS LoadBalancer hostname. Copy the load balancer hostname

---

## 45. Get Argo CD Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d; echo
```
Save the password temporarily.
Open browser --> paste load balancer hostname here --> Login:
Username: admin
Password: generated password
Open:  http://ARGOCD-LOADBALANCER
Your browser may display a certificate warning because this is the default Argo CD TLS setup.

---

## 46. Create Argo CD Application
In Argo CD:
Applications   --> New App
Use:
Application Name:  voting-app

Project:   default

Sync Policy: Manual
Source:
Repository:  https://github.com/harathi-mutyam/voting-app-cicd.git

Revision:  main

Path:   k8s-specifications
Destination:
Cluster:  https://kubernetes.default.svc

Namespace:  voting-app
Click:  Create

---

## 47. Synchronize Argo CD
Open:
Applications  --> voting-app
Click:  SYNC
Then: SYNCHRONIZE
The flow is: 
GitHub

```text
   ↓
```
k8s-specifications

```text
   ↓
```
Argo CD

```text
   ↓
```
EKS

```text
   ↓
```
Voting Application

Open server instance gitbash terminal
Verify:

```bash
kubectl get all -n voting-app
```
And:

```bash
kubectl get pods -n voting-app
```

---

## 48. Demonstrate GitOps
This is an excellent part of your presentation.
Open in browser:GitHub    --> --> voting-app-cicd --> k8s-specifications
--> vote-deployment.yaml
Change: replicas: 1  to:  replicas: 2
Commit directly to main.
Then open Argo CD in browser. 
Click: Refresh
Argo CD should detect that:
Git configuration != Kubernetes configuration
The application becomes:  OutOfSync
Click:   SYNC --> SYNCHRONIZE
Verify:

```bash
kubectl get pods -n voting-app
```
You should see two Vote pods.
The presentation explanation:
GitHub change

```text
      ↓
```
Argo CD detects OutOfSync

```text
      ↓
```
Manual synchronization

```text
      ↓
```
Kubernetes configuration updated

```text
      ↓
```
2 Vote pods running

---

## 49. Install Helm
On the Server EC2:

```bash
helm version
```
If missing:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
Verify:

```bash
helm version
```

---

## 50. Add Prometheus Repository

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
```
Update:

```bash
helm repo update
```
Check:

```bash
helm repo list
```

---

## 51. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```
Verify:

```bash
kubectl get namespaces
```

---

## 52. Install Prometheus + Grafana

```bash
helm install monitoring \
prometheus-community/kube-prometheus-stack \
--namespace monitoring
```
Check:

```bash
helm list -n monitoring
```

---

## 53. Check Monitoring Pods

```bash
kubectl get pods -n monitoring
```
Watch:

```bash
kubectl get pods -n monitoring -w
```
Press:
Ctrl+C
after everything is running.

---

## 54. Check Monitoring Services

```bash
kubectl get svc -n monitoring
```

---

## 55. Expose Grafana

```bash
kubectl patch svc monitoring-grafana \
-n monitoring \
-p '{"spec":{"type":"LoadBalancer"}}'
```
Check:

```bash
kubectl get svc -n monitoring
```
Wait for the external hostname.copy the external hostname 

---

## 56. Get Grafana Password

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" | base64 -d
```
open browser  --> paste the copied Grafana external host name --> Login:
Username: admin

Password: YOUR_GRAFANA_PASSWORD
Open:
http://GRAFANA-LOADBALANCER

---

## 57. Verify Prometheus Data Source
In Grafana: Connections --> Data sources  --> Prometheus  
The internal Prometheus URL should be similar to:
http://monitoring-kube-prometheus-prometheus.monitoring:9090
This is a Kubernetes internal service address.
Do not try to open that internal address directly from your laptop browser.
Grafana uses it from inside the cluster.

---

## 58. Test Prometheus Internally
Check:

```bash
kubectl get svc -n monitoring | grep prometheus
```
Check pods:

```bash
kubectl get pods -n monitoring
```
Test:

```bash
kubectl run curl-test \
-n monitoring \
--rm -it \
--restart=Never \
--image=curlimages/curl \
-- curl -s http://monitoring-kube-prometheus-prometheus.monitoring:9090/-/ready
```
A successful response confirms Prometheus is reachable inside Kubernetes.

---

## 59. Optional — Prometheus Browser Access
If you want to demonstrate Prometheus directly:

```bash
kubectl port-forward \
-n monitoring \
svc/monitoring-kube-prometheus-prometheus \
9090:9090
```
Then open:
http://localhost:9090
Alternatively, for your learning setup, you can expose it:

```bash
kubectl patch svc monitoring-kube-prometheus-prometheus \
-n monitoring \
-p '{"spec":{"type":"LoadBalancer"}}'
```
Then:

```bash
kubectl get svc -n monitoring
```

---

## 60. Grafana Kubernetes Dashboard
In Grafana:
Dashboards
Open the Kubernetes dashboard.
You can demonstrate:
•	Cluster health
•	Nodes
•	Pods
•	CPU
•	Memory
•	Kubernetes workloads
•	Namespace information

---

## 61. Final Verification
EKS

```bash
kubectl get nodes
```
Voting application

```bash
kubectl get all -n voting-app
```
Argo CD

```bash
kubectl get pods -n argocd
```
Monitoring

```bash
kubectl get pods -n monitoring
```
Voting services

```bash
kubectl get svc -n voting-app
```
Argo CD service

```bash
kubectl get svc -n argocd
```
Monitoring services

```bash
kubectl get svc -n monitoring
```

---

## 62. Final Presentation Story
When explaining the project, you can describe it like this:
Developer

```text
   ↓
```
Push code to GitHub

```text
   ↓
```
GitHub Actions

```text
   ↓
```
Python CI

```text
   ↓
```
Node.js CI

```text
   ↓
```
Worker Docker Build

```text
   ↓
```
Gitleaks

```text
   ↓
```
SonarQube

```text
   ↓
```
Docker Build & Push

```text
   ↓
```
Docker Hub

```text
   ↓
```
Terraform-created EKS

```text
   ↓
```
Argo CD

```text
   ↓
```
Kubernetes

```text
   ↓
```
Voting Application

```text
   ↓
```
Prometheus

```text
   ↓
```
Grafana
The three application images are:
harathi2026/voting-vote:latest
harathi2026/voting-result:latest
harathi2026/voting-worker:latest

---

## 63. Cleanup After Presentation
Because AWS resources cost money, clean up after the demonstration.
Delete Voting Application

```bash
kubectl get all -n voting-app
```
Then:

```bash
kubectl delete namespace voting-app
```
Verify:

```bash
kubectl get namespace voting-app
```

---

## 64. Remove Monitoring
Check:

```bash
helm list -n monitoring
```
Uninstall:

```bash
helm uninstall monitoring -n monitoring
```
Delete namespace:

```bash
kubectl delete namespace monitoring
```
Verify:

```bash
kubectl get namespace monitoring
```

---

## 65. Remove Argo CD

```bash
kubectl get namespace argocd
```
Delete:

```bash
kubectl delete namespace argocd
```
Verify:

```bash
kubectl get namespace argocd
```

---

## 66. Destroy EKS Infrastructure
Go to:

```bash
cd ~/voting-app-cicd/terraform
```
Check Terraform state:

```bash
terraform state list
```
Preview:

```bash
terraform plan -destroy
```
Review carefully.
Then:

```bash
terraform destroy --auto-approve
```

---

## 67. Verify Terraform Cleanup

```bash
terraform state list
```
Ideally there should be no remaining managed resources.
Then:

```bash
terraform plan
```
You should see that there is nothing to create/change.
Verify EKS:

```bash
aws eks list-clusters --region eu-north-1
```

---

## 68. Check AWS Console
Verify that the resources created for the project have been removed:
EKS
EC2 worker instances
Load Balancers
VPC
Subnets
Internet Gateway
Security Groups
Terraform-created IAM roles
Do not manually delete Terraform-managed resources before terraform destroy, because that can leave Terraform state inconsistent.

---

## 69. Delete Presentation EC2 Instances
After Terraform has finished destroying the EKS infrastructure, terminate the EC2 instances you created specifically for the demonstration, such as:
voting-app-cicd-runner
sonarqube-server
Server
Be careful not to terminate unrelated AWS resources.

---

## 70. What You Need to Remember for the Next Presentation
The most important change from your first trial is:
You start from the existing repository

```bash
git clone https://github.com/harathi-mutyam/voting-app-cicd.git
```
You do NOT do this anymore

```bash
rm -rf .git
git init
```
You do NOT need these manual image builds

```bash
docker build -t voting-vote:local ./vote
docker build -t voting-result:local ./result
docker build -t voting-worker:local ./worker
```
GitHub Actions does the Docker builds
Docker Build

```text
     ↓
```
Docker Hub
EKS pulls from Docker Hub
EKS

```text
 ↓
```
harathi2026/voting-vote:latest
harathi2026/voting-result:latest
harathi2026/voting-worker:latest
Argo CD handles Kubernetes GitOps
GitHub

```text
 ↓
```
Argo CD

```text
 ↓
```
EKS
Prometheus + Grafana handle monitoring
EKS

```text
 ↓
```
Prometheus

```text
 ↓
```
Grafana

---
This version is the one I would keep as your master presentation document. It assumes the repository is already prepared and removes the unnecessary first-time repository setup and manual local Docker-image testing while retaining the complete infrastructure, CI/CD, GitOps, and monitoring demonstration.
For presetation after creation your project setup:
4-Day Break — Simple Stop, Destroy & Restart Procedure
This procedure is for taking a 4-day break after successfully testing the project and then recreating the environment before the presentation.

## 1. Before Taking the Break
First verify that everything is working in server gitbash terminal:

```bash
kubectl get nodes
kubectl get pods -n voting-app
kubectl get pods -n argocd
kubectl get pods -n monitoring
```
Make sure the following are working:
•	Voting application ,Argo CD ,Prometheus , Grafana , SonarQube , GitHub Actions

---

## 2. Remove Kubernetes Resources
Remove Voting Application

```bash
kubectl delete namespace voting-app
```
Remove Monitoring
Check Helm:

```bash
helm list -n monitoring
```
Uninstall:

```bash
helm uninstall monitoring -n monitoring
```
Delete namespace:

```bash
kubectl delete namespace monitoring
```
Remove Argo CD

```bash
kubectl delete namespace argocd
```

---

## 3. Destroy EKS Infrastructure
Go to Terraform:

```bash
cd ~/voting-app-cicd/terraform
terraform destroy --auto-approve
```
yes
Wait for:
Destroy complete!
Verify:

```bash
terraform state list
```
Check EKS:

```bash
aws eks list-clusters --region eu-north-1
```
Your project EKS cluster should no longer be present.

---

## 4. Stop the EC2 Instances
Go to:
AWS Console --> EC2 --> Instances
Before stopping, note the current:
•	Runner EC2 IP
•	SonarQube EC2 IP
•	Server EC2 IP
•	SonarQube URL
Stop: voting-app-cicd-runner ,sonarqube-server ,Server
Use:
Instance state --> Stop instance

---
On Presentation Day

## 5. Start the EC2 Instances
Go to:
AWS Console --> EC2 --> Instances
Start:  voting-app-cicd-runner , sonarqube-server , Server Ec2 instances

---

## 6. Check New Public IPs
After starting, check:
EC2 --> Instances --> copy new Public IPv4 address
Use the new IPs for SSH.

---

## 7. Start GitHub Actions Runner
SSH into Runner:

```bash
ssh -i Downloads/github-key.pem ubuntu@NEW_RUNNER_IP
```
Then:

```bash
cd ~/actions-runner
```
./run.sh
You should see:
Connected to GitHub
Listening for Jobs
Check GitHub in browser -->Repository --> Settings --> Actions --> Runners
Runner should show: Idle

---

## 8. Check SonarQube
SSH:

```bash
ssh -i Downloads/github-key.pem ubuntu@NEW_SONARQUBE_IP
```
Check:

```bash
docker ps
```
If SonarQube is stopped:

```bash
docker start sonarqube
```
Then:  docker ps
Open:
http://NEW_SONARQUBE_IP:9000

---

## 9. Update SonarQube URL in GitHub
Go to:
GitHub --> Repository --> Settings --> Secrets and variables --> Actions
Update: SONAR_HOST_URL
to: new SonarQube  instance public ip 
http://NEW_SONARQUBE_IP:9000
SONAR_TOKEN normally does not need to change.

---

## 10. Start EKS Again
SSH into Server:

```bash
ssh -i Downloads/github-key.pem ubuntu@NEW_SERVER_IP
```
Check:

```bash
aws --version
terraform version
kubectl version --client
```
Go to Terraform:

```bash
cd ~/voting-app-cicd/terraform
```
Run:

```bash
terraform init
terraform plan
```
Then:

```bash
terraform apply --auto-approve
```
Wait until EKS is created.

---

## 11. Configure kubectl

```bash
aws eks update-kubeconfig \
  --region eu-north-1 \
  --name eks-cluster
```
Check:

```bash
kubectl get nodes
```

---

## 12. Deploy Voting Application

```bash
cd ~/voting-app-cicd
```
Apply Kubernetes files:

```bash
kubectl apply -f k8s-specifications/
```
Check:

```bash
kubectl get pods -n voting-app
kubectl get svc -n voting-app
```
Run and Verify CI Pipeline Before Deployment
  GitHub Access Token & Git Push Procedure
The self-hosted runner uses HTTPS to communicate with GitHub.
For secure GitHub authentication, use a GitHub Personal Access Token (PAT) instead of the GitHub account password.

1 . Configure the Fine-Grained Token
Open the github project repository:
https://github.com/harathi-mutyam/voting-app-cicd
Go to:
(On right side top corner )Profile --> Settings --> Developer settings (on bottom)--> Personal access tokens --> Fine-grained tokens
Click: Generate new token

Token Name : voting-runner
Repository Access  
Select:  Only select repositories
Select: harathi-mutyam/voting-app-cicd
Repository Permissions  : Under Repository permissions, select :  (checkbox ) Contents --> Read and write
 (checkbox) Actions --> Read and Write
Click: Generate token
Copy the generated token and keep it secure.
The token is used instead of the GitHub account password when performing Git operations over HTTPS.

Open new gitbash terminal:

```bash
ssh -i Downloads/github-key.pem ubuntu@voting-app-runner ec2 instance publicip
ls
cd voting-app-cicd
```

Git Push Procedure 
#  Check Git status

```bash
git status
```
#  Add the changes

```bash
git add .
```
#  Commit the changes

```bash
git commit -m "Update CI workflow"
```
If GitHub contains changes that are not present locally:optinal command

```bash
git pull --rebase origin main
```
#  Push the changes to GitHub

```bash
git push origin main
```
When Git asks for credentials:
Username: harathi-mutyam
Password: <GitHub Personal Access Token>
Note: Use the GitHub Personal Access Token instead of the GitHub account password.
#  Verify

```bash
git status
```
Expected:
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean


Open browser --> GitHub --> voting-app-cicd --> Actions  -->
we should see: Voting App CI
The workflow should execute in this order:
 


---

## 13. Install Argo CD Again

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
--server-side \
--force-conflicts \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Check:

```bash
kubectl get pods -n argocd
```
Then expose argocd-server as a LoadBalancer as in your presentation setup.

---

## 14. Install Monitoring Again
Add Helm repository:

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
```
Update:

```bash
helm repo update
```
Create namespace:

```bash
kubectl create namespace monitoring
```
Install:

```bash
helm install monitoring \
prometheus-community/kube-prometheus-stack \
--namespace monitoring
```
Check:

```bash
kubectl get pods -n monitoring
```
Expose Grafana:

```bash
kubectl patch svc monitoring-grafana \
-n monitoring \
-p '{"spec":{"type":"LoadBalancer"}}'
```
Check:

```bash
kubectl get svc -n monitoring
```

---

## 15. Final Presentation Check
Before starting the presentation:

```bash
kubectl get nodes
kubectl get pods -n voting-app
kubectl get pods -n argocd
kubectl get pods -n monitoring
```
Verify:
•	✅ GitHub Actions Runner — Idle
•	✅ SonarQube — accessible
•	✅ GitHub Actions — working
•	✅ Docker Hub images — available
•	✅ EKS nodes — Ready
•	✅ Voting application — accessible
•	✅ Argo CD — accessible
•	✅ Prometheus — running
•	✅ Grafana — accessible
Simple Flow to Remember
AFTER TRIAL

```text
    ↓
```
Delete Kubernetes workloads

```text
    ↓
```

```bash
terraform destroy
```

```text
    ↓
```
EKS deleted

```text
    ↓
```
STOP EC2 instances

```text
    ↓
```
4 DAYS BREAK

```text
    ↓
```
START EC2 instances

```text
    ↓
```
Check NEW IPs

```text
    ↓
```
Start GitHub Runner

```text
    ↓
```
Start/check SonarQube

```text
    ↓
```

```bash
terraform apply
```

```text
    ↓
```
EKS created

```text
    ↓
```
Deploy Voting App

```text
    ↓
```
Install Argo CD

```text
    ↓
```
Install Prometheus + Grafana

```text
    ↓
```
PRESENTATION
