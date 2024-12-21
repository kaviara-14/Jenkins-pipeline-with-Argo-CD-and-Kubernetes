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
     -  **Configure Sonar Server in Manage Jenkins :** Goto your Sonarqube Server, < Your Public IP >:9000, generate a token. In Jenkins, Goto Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text. (Put copied token)
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

### 1. Install AWS CLI, Kubectl, eksctl and helm chart
```bash
# Install AWS CLI v2 and configure
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure

# Install kubectl
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install Helm Chart - Use the following script to install the helm chart 
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### 2 . Creating an AWS EKS cluster using eksctl
Now in this step, we are going to create AWS EKS cluster using eksctl

```bash
# You need the following to run the eksctl command
eksctl create cluster --name eks-netflix-1 --version 1.24 --region us-east-1 --nodegroup-name worker-nodes --node-type t2.medium --nodes 1 --nodes-min 1 --nodes-max 1
aws eks update-kubeconfig --region us-east-1 --name eks-netflix-1
kubectl get nodes

# Create IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${EKS_CLUSTER_NAME} \
    --approve

# Download IAM policy for the AWS Load Balancer Controller
curl -fsSL -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy.json

# Create a IAM role and ServiceAccount for the AWS Load Balancer controller using eksctl tool
eksctl create iamserviceaccount \
    --cluster=${EKS_CLUSTER_NAME} \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve \
    --region ${AWS_REGION}

# Install the helm chart by specifying the chart values serviceAccount.create=false and serviceAccount.name=aws-load-balancer-controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=${EKS_CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller

# Install ArgoCD
kubectl create namespace argocd 
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml

# By default, argocd-server is not publically exposed. In this scenario, we will use a Load Balancer to make it usable, get the url
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get the Load balancer DNS
export ARGOCD_SERVER=`kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
echo $ARGOCD_SERVER
export ARGO_PWD=`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
echo $ARGO_PWD

```
### Step 3 : Configure Argocd
* Take the LoadBalancer link and open it in your browser.After installing ArgoCD, you need to set up your GitHub repository as a source for your application deployment. This typically involves configuring the connection to your repository and defining the source for your ArgoCD application.
* Once you configured you have now successfully deployed an application using Argo CD.Argo CD is a Kubernetes controller, responsible for continuously monitoring all running applications and comparing their live state to the desired state specified in the Git repository.



![Screenshot 2024-12-20 070052](https://github.com/user-attachments/assets/848ce0a2-00ee-4f56-9851-4b86056784e5)


---

