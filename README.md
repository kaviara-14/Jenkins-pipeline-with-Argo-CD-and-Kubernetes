# Jenkins-pipeline-with-Argo-CD-and-Kubernetes

This project demonstrates the implementation of a Continuous Integration and Continuous Delivery (CI/CD) pipeline using **Jenkins**, **Argo CD**, and **Kubernetes**. It deploys a Spring Boot application to a Kubernetes cluster.

<img src='https://user-images.githubusercontent.com/43399466/228301952-abc02ca2-9942-4a67-8293-f76647b6f9d8.png'>

---

## Setup Instructions

### EC2 Instance Configuration
Launch an Ec2 Instance with AMIs as Ubuntu and select Instance Type as t2.medium. Create new Key Pair and Create a new Security Group with traffic allowed from ssh, http and https.Add Security inbound rule for port 8010.

``` bash
  # Clone the repository and build the Spring Boot applicationn
  git clone
  cd Jenkins-pipeline-with-Argo-CD-and-Kubernetes/spring-boot-app
  sudo apt update
  sudo apt install maven
  mvn clean package
  mvn -v
  # Install Docker
  sudo apt update
  sudo apt install docker.io
  sudo usermod -aG docker ubuntu
  sudo chmod 666 /var/run/docker.sock
  sudo systemctl restart docker
  # Build and Run the container
  docker build -t ultimate-cicd-pipeline:v1 .
  docker run -d -p 8010:8080 -t ultimate-cicd-pipeline:v1

```
---

# Continuous Integration
### 1. Install Jenkins
Follow the steps for installing Jenkins on the EC2 instance

  ``` bash
  # Install Jenkins on the EC2 instance
  sudo apt update
  sudo apt install openjdk-11-jre -y
  java -version
  curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null
  
  echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null
  
  sudo apt-get update
  sudo apt-get install jenkins
  sudo service jenkins start
  # Retrieve the initial admin password
  cat /var/lib/jenkins/secrets/initialAdminPassword

  ```

### 2. Install and Configure SonarQube
SonarQube ensures high-quality code and identifies bugs during static analysis. Ensure port 9000 is open in your EC2 security group.

``` bash
    # Launch a SonarQube container:
    docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
  ```

### 3. Jenkins Pipeline Configuration
 -  Install the required JenkinsPlugins like docker Pipeline and SonarQube Scanner
 -  Configure credentials in Jenkins
     -  **SonarQube Token:** Generate a token in SonarQube and add it to Jenkins under Manage Jenkins > Credentials.
     -  **GitHub Token:** Generate a personal access token in GitHub and add it to Jenkins.
     -  **Docker Hub Credentials:** Add your Docker Hub username and password to Jenkins.
  
 -  Create a new pipeline
     - Navigate to New Item in Jenkins.
     - Select Pipeline and configure the pipeline with your GitHub repository.
     - Specify the path to your Jenkinsfile.

### Build and Deploy the Application

- Build your Java application in the Jenkins pipeline.
- Analyze code quality using SonarQube.
- Push the built Docker image to Docker Hub.
- Update the deployment manifest with the latest Docker image.

This way, we completed CI (Continuous Integration) Part. Java application is built, SonarQube completed static code analysis and the latest image is created, push to DockerHub and updated Manifest repository with the latest image

![Screenshot 2024-12-19 192237](https://github.com/user-attachments/assets/86427b8d-fbcf-4be9-8fa6-45b645bb496d)
![Screenshot 2024-12-19 192332](https://github.com/user-attachments/assets/57c41126-7b77-4c5f-b462-c2147435545b)
![Screenshot 2024-12-19 192356](https://github.com/user-attachments/assets/d9ec2b80-5f25-45ee-bf6d-453a4e2fae4e)

---

# Continuous Delivery

### 1. Minikube Setup
Minikube enables setting up and running a single-node Kubernetes cluster locally. This is ideal for testing applications before production deployment.

  ```bash
  # Steps to Install Minikube
  sudo apt-get update
  sudo apt-get install docker.io -y
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube
  sudo usermod -aG docker $USER && newgrp docker
  sudo reboot
  minikube start --driver=docker
  
  ```
### 2. Kubectl Installation
Kubectl is a command-line tool to interact with Kubernetes clusters.
  ```bash
    # Steps to Install Minikube
    curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin
    kubectl version
  
  ```

### 3. Install AgroCD
Install Argo CD in your Kubernetes cluster. You can use the official manifests.

  ``` bash
  kubectl create namespace argocd
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  kubectl get pods
  kubectl get svc
  kubectl edit svc example-argocd-server  # Change the Cluster Ip to NodePort
 ```

Access: Expose the Argo CD API server. For a quick setup, use port forwarding:
  ```bash
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```

We will use the Argo CD web interface to run sprint-boot-app.Access the Argo CD web interface and set up your GitHub repository manifest and Kubernetes cluster and sync


![Screenshot 2024-12-20 070052](https://github.com/user-attachments/assets/848ce0a2-00ee-4f56-9851-4b86056784e5)




