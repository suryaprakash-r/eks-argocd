# **AWS EKS Cluster & ArgoCD Setup Guide**

## **1. Configure AWS Credentials**

```sh
aws configure
```

## **2. Create EKS Cluster**

```sh
eksctl create cluster --name blue-green-cluster --region ap-south-1 --nodes 2
```

## **3. Verify Cluster Setup**

```sh
kubectl get nodes
```

## **4. Install ArgoCD in Kubernetes**

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## **5. Expose ArgoCD Server**

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## **6. Retrieve ArgoCD Admin Password**

```sh
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

# **CI/CD Pipeline Setup: GitHub → Jenkins → ECR → ArgoCD → EKS**

## **1. Jenkins Pipeline Setup**

### **Step 1: Configure Jenkins with Required Plugins**

- Install **Pipeline**, **Git**, **Docker**, and **Kubernetes CLI** plugins.
- Add AWS credentials in Jenkins for ECR authentication.

### **Step 2: Create a Jenkins Pipeline**

#### **1. Blog-App Pipeline**

```groovy
pipeline {
    agent any
    environment {
        MAVEN_HOME = "/opt/maven"
        PATH = "$PATH:$MAVEN_HOME/bin"
        AWS_REGION = "ap-south-1"
        ECR_REPO = "<account-id>.dkr.ecr.${AWS_REGION}.amazonaws.com/devops"
        EMAIL_RECIPIENTS = "your-mail@gmail.com"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/suryaprakash-r/blog-app.git'
            }
        }
        stage('Maven Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                    sh 'mv target/*.war target/blog.war'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                    sh "docker build -t ${ECR_REPO}:${commitId} ."
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    sh "docker push ${ECR_REPO}:${commitId}"
                }
            }
        }
    }
}
```

---

## **2. ArgoCD Pipeline**

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/suryaprakash-r/eks-argocd.git'
            }
        }
        stage('Update K8s Deployment YAML') {
            steps {
                sh "sed -i 's|image:.*|image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/devops:'\"${env.COMMIT_ID}\"|' blue-green-deployment.yaml"
                sh 'git add blue-green-deployment.yaml'
                sh 'git commit -m "Updated image tag to ${env.COMMIT_ID}"'
                sh 'git push origin main'
            }
        }
    }
}
```

---

## **3. Create an ArgoCD Application**

### **3.1 Create Project**

```sh
ArgoCD UI -> Settings -> Project
```

### **3.2 Create Repository**

```sh
ArgoCD UI -> Settings -> Repository -> Add Git Source
```

### **3.3 Create ArgoCD Application**

```sh
New Application -> Enter details manually (Project Name -> Cluster details -> Sync policy -> GitHub manifest file (Branch))
```

---

## **4. View Deployment in ArgoCD**

Now you can view your ArgoCD deployment, service, Nginx Ingress, and other applications in your ArgoCD UI.

---

This guide provides a step-by-step workflow to set up an **AWS EKS cluster**, deploy applications using **ArgoCD**, and automate deployments with a **Jenkins CI/CD pipeline**. 🚀

