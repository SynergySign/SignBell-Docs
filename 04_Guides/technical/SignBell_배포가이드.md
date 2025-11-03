# SignBell 배포 가이드 명세서

본 문서는 **SignBell 플랫폼**의 AWS EKS 기반 배포 시스템 구축 및 운영을 위한 **종합 배포 가이드 명세서**입니다.

* **작성자:** [신동준](https://github.com/sdj3959)
* **문서 버전:** v1.0.0
* **최종 수정일:** 2025.11.03

**대상 독자:**
- **DevOps 엔지니어**: 인프라 구축 및 CI/CD 파이프라인 설정
- **백엔드 개발자**: 배포 환경 이해 및 운영 지원
- **프론트엔드 개발자**: 배포 프로세스 이해
- **신규 합류자**: SignBell 플랫폼의 배포 아키텍처 이해

---

## 📋 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [사전 준비](#2-사전-준비)
3. [인프라 구축](#3-인프라-구축)
4. [수동 배포 가이드](#4-수동-배포-가이드)
5. [CI/CD 자동화](#5-cicd-자동화)
6. [GitHub 워크플로우 및 브랜치 전략](#6-github-워크플로우-및-브랜치-전략)
7. [모니터링 및 운영](#7-모니터링-및-운영)
8. [트러블슈팅](#8-트러블슈팅)
9. [FAQ 및 예상 질문](#9-faq-및-예상-질문)

---

## 1. 아키텍처 개요

### 1.1 전체 구조도

```
사용자
  ↓
Route 53 (DNS)
  ↓
┌─────────────────┬─────────────────────────────┐
│                 │                             │
▼                 ▼                             ▼
CloudFront      ALB                         ALB
    +         (api.signbell.app)        (ai.signbell.app)
   S3            ↓                           ↓
(Frontend)    Backend Pods              AI Server Pods
              (Spring Boot)             (FastAPI)
                  ↓                           ↓
                RDS (MariaDB)           Janus WebRTC
```

### 1.2 기술 스택

| 구분 | 기술 |
|------|------|
| **Frontend** | React 19.1.x + Vite |
| **Backend** | Spring Boot 3.5.x + Java 17 |
| **AI Server** | FastAPI + Python 3.10 + Janus WebRTC |
| **Database** | MariaDB 10.11.x (RDS) |
| **Container** | Docker + Kubernetes (EKS) |
| **CI/CD** | Jenkins |
| **Infra** | AWS (EKS, ECR, S3, CloudFront, ALB, RDS) |

### 1.3 배포 흐름

```
코드 Push (GitHub)
  ↓
Jenkins Webhook 트리거
  ↓
┌─────────────┬─────────────┬─────────────┐
│  Frontend   │   Backend   │  AI Server  │
│  npm build  │ Gradle build│ Docker build│
│     ↓       │      ↓      │      ↓      │
│  S3 Upload  │  ECR Push   │  ECR Push   │
│     ↓       │      ↓      │      ↓      │
│ CloudFront  │ EKS Deploy  │ EKS Deploy  │
└─────────────┴─────────────┴─────────────┘
```

---

## 2. 사전 준비

### 2.1 필수 도구 설치

#### macOS
```bash
# Homebrew 설치
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 필수 도구 설치
brew install awscli kubectl eksctl helm docker
```

#### Windows
```powershell
# Chocolatey 설치 (관리자 권한 PowerShell)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# 필수 도구 설치
choco install awscli kubectl eksctl kubernetes-helm docker-desktop
```

### 2.2 AWS 계정 설정

```bash
# AWS CLI 설정
aws configure --profile signbell
# Access Key ID: [IAM 사용자 키]
# Secret Access Key: [IAM 사용자 시크릿]
# Default region: ap-northeast-2
# Default output: json

# 설정 확인
aws sts get-caller-identity --profile signbell
```

---

## 3. 인프라 구축

### 3.1 VPC 및 네트워크 생성

```bash
# VPC 생성 (AWS 콘솔 또는 eksctl이 자동 생성)
# - CIDR: 10.0.0.0/16
# - Public Subnet: 2개 (각 AZ)
# - Private Subnet: 2개 (각 AZ)
# - NAT Gateway: 1개 (비용 절감)
```

### 3.2 EKS 클러스터 생성

**cluster.yaml 작성**:
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: signbell-cluster
  region: ap-northeast-2
  version: "1.29"

managedNodeGroups:
  - name: ng-signbell
    instanceType: t3.small
    minSize: 2
    maxSize: 3
    desiredCapacity: 2
    volumeSize: 30
    ssh:
      allow: true
      publicKeyName: signbell-eks-key
    privateNetworking: true
```

**클러스터 생성**:
```bash
# 키 페어 생성 (AWS 콘솔에서 미리 생성)
# 클러스터 생성 (15-20분 소요)
eksctl create cluster -f cluster.yaml --profile signbell

# kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name signbell-cluster --profile signbell

# 노드 확인
kubectl get nodes
```

### 3.3 ECR 리포지토리 생성

```bash
# Backend ECR
aws ecr create-repository \
  --repository-name signbell-backend \
  --region ap-northeast-2 \
  --profile signbell

# AI Server ECR
aws ecr create-repository \
  --repository-name signbell-ai \
  --region ap-northeast-2 \
  --profile signbell

# URI 확인 및 메모
aws ecr describe-repositories --region ap-northeast-2 --profile signbell
```

### 3.4 RDS (MariaDB) 생성

```bash
# AWS 콘솔에서 생성
# - 엔진: MariaDB 10.11
# - 인스턴스: db.t3.micro
# - 스토리지: 20GB gp3
# - VPC: EKS와 동일
# - 서브넷: Private
# - 보안 그룹: EKS 워커 노드에서 3306 포트 허용

# 엔드포인트 확인
aws rds describe-db-instances \
  --db-instance-identifier signbell-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text \
  --profile signbell
```

### 3.5 ALB Ingress Controller 설치

```bash
# IAM OIDC Provider 생성
eksctl utils associate-iam-oidc-provider \
  --cluster signbell-cluster \
  --region ap-northeast-2 \
  --approve \
  --profile signbell

# IAM Policy 생성
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json \
  --profile signbell

# ServiceAccount 생성
eksctl create iamserviceaccount \
  --cluster=signbell-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::801490935747:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve \
  --region=ap-northeast-2 \
  --profile signbell

# Helm으로 ALB Controller 설치
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=signbell-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-northeast-2

# 설치 확인
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 3.6 보안 그룹 설정

```bash
# EKS 워커 노드 보안 그룹 찾기
aws ec2 describe-security-groups \
  --filters "Name=tag:Name,Values=*eks-cluster-sg*" \
  --query 'SecurityGroups[0].GroupId' \
  --output text \
  --profile signbell

# WebRTC UDP 포트 열기 (AWS 콘솔에서 수동 설정)
# - UDP 10000-20000 (WebRTC Media)
# - UDP 49152-65535 (Dynamic Ports)
# - 소스: 0.0.0.0/0
```

---

## 4. 수동 배포 가이드

### 4.1 Backend 수동 배포

#### 1단계: 로컬 빌드
```bash
cd backend

# Gradle 빌드
./gradlew clean build -x test

# 빌드 결과 확인
ls -lh build/libs/*.jar
```

#### 2단계: Docker 이미지 빌드
```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 --profile signbell | \
  docker login --username AWS --password-stdin 801490935747.dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드 (Apple Silicon 주의!)
docker build --platform linux/amd64 -t signbell-backend:latest .

# 이미지 태그
docker tag signbell-backend:latest \
  801490935747.dkr.ecr.ap-northeast-2.amazonaws.com/signbell-backend:v1.0

# ECR 푸시
docker push 801490935747.dkr.ecr.ap-northeast-2.amazonaws.com/signbell-backend:v1.0
```

#### 3단계: Kubernetes Secret 생성
```bash
# Secret 생성 (최초 1회만)
kubectl create secret generic signbell-secrets \
  --from-literal=db-password='mariadb1234' \
  --from-literal=jwt-secret='kq2B8xv4zJ9L1t6Q3w8Y2u5R7o0M3n6B9c2E5h8J1k4=' \
  --from-literal=kakao-client-id='b0a11cd4478b43be83addcc4a7ff9516' \
  --from-literal=kakao-client-secret='qP6a7Er2xp1Qy3rT6hToiVZ6VP740mGE' \
  --from-literal=api-service-key='8e94127e-9ea0-4f85-a4af-f36bd7110bb0' \
  --namespace=default

# ConfigMap 생성
kubectl create configmap signbell-config \
  --from-literal=SERVER_PORT='8443' \
  --from-literal=DB_URL='jdbc:mariadb://signbell-db.c9g2mk6gms0l.ap-northeast-2.rds.amazonaws.com:3306/signbell' \
  --from-literal=DB_USERNAME='admin' \
  --from-literal=COOKIE_DOMAIN='.signbell.app' \
  --namespace=default

# 확인
kubectl get secrets
kubectl get configmaps
```

#### 4단계: Deployment 배포
```bash
# Deployment 적용
kubectl apply -f k8s/backend-deployment.yaml

# Service 적용
kubectl apply -f k8s/backend-service.yaml

# Pod 상태 확인
kubectl get pods -l app=signbell-backend

# 로그 확인
kubectl logs -f deployment/signbell-backend-deployment
```

### 4.2 AI Server 수동 배포

#### 1단계: Docker 이미지 빌드
```bash
cd ai-server

# ECR 로그인 (이미 했다면 생략)
aws ecr get-login-password --region ap-northeast-2 --profile signbell | \
  docker login --username AWS --password-stdin 801490935747.dkr.ecr.ap-northeast-2.amazonaws.com

# 이미지 빌드
docker build --platform linux/amd64 -t signbell-ai:latest .

# 이미지 태그
docker tag signbell-ai:latest \
  801490935747.dkr.ecr.ap-northeast-2.amazonaws.com/signbell-ai:v1.0

# ECR 푸시
docker push 801490935747.dkr.ecr.ap-northeast-2.amazonaws.com/signbell-ai:v1.0
```

#### 2단계: Deployment 배포
```bash
# Deployment 적용
kubectl apply -f k8s/ai-deployment.yaml

# Service 적용
kubectl apply -f k8s/ai-service.yaml

# Pod 상태 확인
kubectl get pods -l app=signbell-ai

# 로그 확인
kubectl logs -f deployment/signbell-ai-deployment
```

### 4.3 Frontend 수동 배포

#### 1단계: 빌드
```bash
cd frontend

# 의존성 설치
npm install

# 프로덕션 빌드
NODE_ENV=production npm run build

# 빌드 결과 확인
ls -lh dist/
```

#### 2단계: S3 업로드
```bash
cd dist

# S3 동기화
aws s3 sync . s3://www.signbell.app --delete --profile signbell

# 업로드 확인
aws s3 ls s3://www.signbell.app/ --profile signbell
```

#### 3단계: CloudFront 캐시 무효화
```bash
# 캐시 무효화
aws cloudfront create-invalidation \
  --distribution-id E1BB9LGYVUR99C \
  --paths "/*" \
  --profile signbell

# 무효화 상태 확인
aws cloudfront list-invalidations \
  --distribution-id E1BB9LGYVUR99C \
  --profile signbell
```

### 4.4 Ingress 배포 (ALB 생성)

```bash
# Ingress 적용
kubectl apply -f k8s/ingress.yaml

# ALB 생성 확인 (2-3분 소요)
kubectl get ingress

# ALB DNS 확인
kubectl get ingress signbell-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# ALB 상태 확인
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-default-signbell`)].{Name:LoadBalancerName,DNS:DNSName,State:State.Code}' \
  --output table \
  --profile signbell
```

---

## 5. CI/CD 자동화 (Jenkins)

### 5.0 Jenkins 자동화 개요

**목표**: Git Push 시 Frontend, Backend, AI Server 자동 배포

**배포 흐름**:
```
Git Push (main 브랜치)
  ↓
GitHub Webhook 트리거
  ↓
Jenkins Pipeline 실행
  ↓
┌─────────────┬─────────────┬─────────────┐
│  Frontend   │   Backend   │  AI Server  │
│  npm build  │ Gradle build│ Docker build│
│     ↓       │      ↓      │      ↓      │
│  S3 Upload  │  ECR Push   │  ECR Push   │
│     ↓       │      ↓      │      ↓      │
│ CloudFront  │ EKS Deploy  │ EKS Deploy  │
└─────────────┴─────────────┴─────────────┘
```

### 5.1 Jenkins 서버 구축

#### 1단계: EC2 인스턴스 생성
```bash
# AWS 콘솔에서 생성
# - AMI: Amazon Linux 2023
# - 인스턴스 타입: t3.small (t3.micro는 메모리 부족)
# - 키 페어: signbell-jenkins-key
# - 보안 그룹: 
#   - SSH (22): 내 IP
#   - HTTP (8080): 내 IP
#   - HTTPS (443): 0.0.0.0/0
# - IAM 역할: signbell-jenkins-role (ECR, S3, EKS 권한)
```

#### 2단계: Docker 설치
```bash
# EC2 접속
ssh -i ~/.ssh/signbell-jenkins-key.pem ec2-user@<Jenkins-EC2-IP>

# Docker 설치
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# 재접속 후 확인
docker --version
```

#### 3단계: Jenkins 컨테이너 실행
```bash
# Jenkins 볼륨 생성
docker volume create jenkins_home

# Jenkins 실행
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  --restart=on-failure \
  --user root \
  jenkins/jenkins:lts-jdk17

# 초기 비밀번호 확인
docker logs jenkins | grep -A 5 "Please use the following password"

# 브라우저에서 접속: http://<Jenkins-EC2-IP>:8080
```

#### 4단계: Jenkins 플러그인 설치
```
Jenkins 관리 > Plugins > Available plugins

필수 플러그인:
- Amazon ECR
- Kubernetes CLI
- Docker Pipeline
- Blue Ocean (선택)
```

#### 5단계: Jenkins 컨테이너 도구 설치

Jenkins 컨테이너 내부에 빌드에 필요한 도구들을 설치합니다.

**전체 설치 스크립트 (한 번에 실행)**:
```bash
# SSH로 Jenkins EC2 접속
ssh -i ~/.ssh/signbell-jenkins-key.pem ec2-user@<Jenkins-EC2-IP>

# 0. Docker 설치 (가장 먼저! 필수!)
echo "=== Docker 설치 중... ==="
docker exec -u root jenkins bash -c 'apt-get update && apt-get install -y docker.io && docker --version'
docker exec -u root jenkins bash -c 'chmod 666 /var/run/docker.sock'

# 1. Node.js 20 설치
echo "=== Node.js 20 설치 중... ==="
docker exec -u root jenkins bash -c 'curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && apt-get install -y nodejs && node --version && npm --version'

# 2. Gradle 8.5 설치
echo "=== Gradle 8.5 설치 중... ==="
docker exec -u root jenkins bash -c 'cd /tmp && wget -q https://services.gradle.org/distributions/gradle-8.5-bin.zip && unzip -q gradle-8.5-bin.zip -d /opt && ln -sf /opt/gradle-8.5/bin/gradle /usr/local/bin/gradle && rm gradle-8.5-bin.zip && gradle --version'

# 3. Python 3.10.11 설치 (10-15분 소요)
echo "=== Python 3.10.11 설치 중 (10-15분 소요)... ==="
docker exec -u root jenkins bash -c 'apt-get update && apt-get install -y build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev wget libbz2-dev && cd /tmp && wget https://www.python.org/ftp/python/3.10.11/Python-3.10.11.tgz && tar -xf Python-3.10.11.tgz && cd Python-3.10.11 && ./configure --enable-optimizations && make -j $(nproc) && make altinstall && ln -sf /usr/local/bin/python3.10 /usr/local/bin/python3 && ln -sf /usr/local/bin/pip3.10 /usr/local/bin/pip3 && cd / && rm -rf /tmp/Python-3.10.11* && python3 --version && pip3 --version'

# 4. kubectl 설치
echo "=== kubectl 설치 중... ==="
docker exec -u root jenkins bash -c 'curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && mv kubectl /usr/local/bin/ && kubectl version --client'

# 5. AWS CLI 설치
echo "=== AWS CLI 설치 중... ==="
docker exec -u root jenkins bash -c 'apt-get update && apt-get install -y awscli && aws --version'

# 6. 최종 확인
echo "=== 설치 완료! 버전 확인 ==="
docker exec jenkins bash -c 'echo "Docker: $(docker --version)" && echo "Node.js: $(node --version)" && echo "npm: $(npm --version)" && echo "Gradle: $(gradle --version | head -3)" && echo "Python: $(python3 --version)" && echo "pip: $(pip3 --version)" && echo "kubectl: $(kubectl version --client --short 2>/dev/null || kubectl version --client)" && echo "AWS CLI: $(aws --version)"'

# 7. Docker 작동 테스트
echo "=== Docker 작동 테스트 ==="
docker exec jenkins docker ps
```

**설치 완료 후 예상 출력**:
```
Docker: Docker version 20.10.x, build xxxxx
Node.js: v20.x.x
npm: 10.x.x
Gradle: 8.5
Python: Python 3.10.11
pip: pip 23.x.x from /usr/local/lib/python3.10/site-packages/pip (python 3.10)
kubectl: Client Version: v1.28.x
AWS CLI: aws-cli/1.x.x Python/3.x.x Linux/x.x.x
```

**⚠️ 중요**: Docker 설치를 가장 먼저 해야 합니다! Docker 없이는 이미지 빌드가 불가능합니다.

#### 6단계: Jenkins Credentials 설정
```
Jenkins 관리 > Credentials > System > Global credentials

1. AWS Credentials
   - Kind: AWS Credentials
   - ID: aws-credentials
   - Access Key ID: [IAM 사용자 키]
   - Secret Access Key: [IAM 사용자 시크릿]

2. GitHub Credentials (Git SCM용)
   - Kind: Username with password
   - Username: [GitHub 사용자명]
   - Password: [GitHub Personal Access Token]
   - ID: github-credentials
   - Description: GitHub Access for SCM

3. Kubeconfig
   - Kind: Secret file
   - ID: kubeconfig
   - File: ~/.kube/config 업로드
```

**GitHub Personal Access Token 생성**:
```
GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)
Generate new token (classic)

권한 선택:
☑ repo (전체)
☑ admin:repo_hook

Generate token → 복사 (ghp_로 시작)
```

### 5.2 Jenkins Pipeline 구성

#### Jenkinsfile (프로젝트 루트)
```groovy
pipeline {
    agent none

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REGISTRY = '801490935747.dkr.ecr.ap-northeast-2.amazonaws.com'
        BACKEND_REPO = 'signbell-backend'
        AI_REPO = 'signbell-ai'
        S3_BUCKET = 'www.signbell.app'
        CF_DISTRIBUTION = 'E1BB9LGYVUR99C'
    }

    stages {
        stage('Build Frontend') {
            agent { label 'fe-agent' }
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'NODE_ENV=production npm run build'
                }
            }
        }

        stage('Deploy Frontend') {
            agent { label 'fe-agent' }
            steps {
                dir('frontend/dist') {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh "aws s3 sync . s3://${S3_BUCKET} --delete"
                        sh "aws cloudfront create-invalidation --distribution-id ${CF_DISTRIBUTION} --paths '/*'"
                    }
                }
            }
        }

        stage('Build Backend') {
            agent { label 'be-agent' }
            steps {
                dir('backend') {
                    sh './gradlew clean build -x test'
                }
            }
        }

        stage('Push Backend Image') {
            agent { label 'be-agent' }
            steps {
                dir('backend') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                            sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker build --platform linux/amd64 -t ${BACKEND_REPO}:${BUILD_NUMBER} .
                            docker tag ${BACKEND_REPO}:${BUILD_NUMBER} ${ECR_REGISTRY}/${BACKEND_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy Backend') {
            agent { label 'be-agent' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    export KUBECONFIG=\$KUBECONFIG_FILE
                    kubectl set image deployment/signbell-backend-deployment \
                      signbell-backend=${ECR_REGISTRY}/${BACKEND_REPO}:${BUILD_NUMBER} \
                      -n default
                    kubectl rollout status deployment/signbell-backend-deployment -n default --timeout=5m
                    """
                }
            }
        }

        stage('Push AI Image') {
            agent { label 'ai-agent' }
            steps {
                dir('ai-server') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                            sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker build --platform linux/amd64 -t ${AI_REPO}:${BUILD_NUMBER} .
                            docker tag ${AI_REPO}:${BUILD_NUMBER} ${ECR_REGISTRY}/${AI_REPO}:${BUILD_NUMBER}
                            docker push ${ECR_REGISTRY}/${AI_REPO}:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy AI Server') {
            agent { label 'ai-agent' }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    export KUBECONFIG=\$KUBECONFIG_FILE
                    kubectl set image deployment/signbell-ai-deployment \
                      signbell-ai=${ECR_REGISTRY}/${AI_REPO}:${BUILD_NUMBER} \
                      -n default
                    kubectl rollout status deployment/signbell-ai-deployment -n default --timeout=5m
                    """
                }
            }
        }
    }

    post {
        always {
            deleteDir()
        }
    }
}
```

### 5.3 GitHub Webhook 설정

```bash
# GitHub Repository > Settings > Webhooks > Add webhook

Payload URL: http://<Jenkins-EC2-IP>:8080/github-webhook/
Content type: application/json
Events: Just the push event
Active: ✓
```

### 5.4 Jenkins 컨테이너 도구 설치

#### 필수 도구 설치

Jenkins 컨테이너에서 빌드를 실행하려면 다음 도구가 필요합니다:

```bash
# Jenkins EC2에 SSH 접속 후 실행

# 1. Node.js 20 설치
docker exec -it -u root jenkins bash -c 'apt-get update && curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && apt-get install -y nodejs'

# 2. Gradle 8.5 설치
docker exec -it -u root jenkins bash -c 'wget -q https://services.gradle.org/distributions/gradle-8.5-bin.zip && unzip -q gradle-8.5-bin.zip -d /opt && ln -sf /opt/gradle-8.5/bin/gradle /usr/local/bin/gradle && rm gradle-8.5-bin.zip'

# 3. Python 3.10.11 설치 (10-15분 소요)
docker exec -it -u root jenkins bash -c 'apt-get install -y build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev libsqlite3-dev wget libbz2-dev && cd /tmp && wget https://www.python.org/ftp/python/3.10.11/Python-3.10.11.tgz && tar -xf Python-3.10.11.tgz && cd Python-3.10.11 && ./configure --enable-optimizations && make -j $(nproc) && make altinstall && ln -sf /usr/local/bin/python3.10 /usr/local/bin/python3 && ln -sf /usr/local/bin/pip3.10 /usr/local/bin/pip3 && cd / && rm -rf /tmp/Python-3.10.11*'

# 4. kubectl 설치
docker exec -it -u root jenkins bash -c 'curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && mv kubectl /usr/local/bin/'

# 5. 설치 확인
docker exec -it jenkins bash -c 'node --version && npm --version && gradle --version && python3 --version && kubectl version --client'
```

**설치 완료 후 예상 출력**:
```
v20.x.x
10.x.x
Gradle 8.5
Python 3.10.11
Client Version: v1.28.x
```

### 5.5 파이프라인 전략

#### 통합 파이프라인 (현재 방식 - 권장)

**구조**:
```
SignBell-App (모노레포)
├── frontend/
├── backend/
└── Jenkinsfile (통합)

SignBell-FASTAPI (별도 레포)
└── AI Server (Jenkinsfile에서 checkout)
```

**장점**:
- ✅ 한 번의 배포로 전체 스택 업데이트
- ✅ 프론트-백엔드 동기화 보장
- ✅ 관리 포인트 단순화
- ✅ 배포 시간 단축 (병렬 실행 가능)

**단점**:
- ❌ 한 부분 실패 시 전체 배포 실패
- ❌ AI 서버만 업데이트 시에도 전체 빌드

#### 조건부 실행 (향후 고려)

배포 빈도가 높아지면 `when` 조건 추가:

```groovy
stage('Build Frontend') {
    when {
        anyOf {
            changeset "frontend/**"
            expression { params.FORCE_BUILD == 'true' }
        }
    }
    steps {
        // ...
    }
}
```

**적용 시기**:
- 배포 빈도가 하루 10회 이상
- 빌드 시간이 30분 이상
- 팀 규모 확대 (5명 이상)

#### 완전 분리 (비추천)

**이유**:
- ❌ 관리 복잡도 급증
- ❌ 프론트-백엔드 동기화 문제
- ❌ 배포 실패 시 디버깅 어려움
- ❌ 3개 파이프라인 관리 부담

### 5.6 멀티 브랜치 파이프라인 설정

#### 브랜치 전략

```
deploy (스테이징) → dev (개발) → main (프로덕션)
```

**브랜치별 역할**:
- **deploy**: 테스트 배포 (프로덕션과 동일 환경)
- **dev**: 개발 환경
- **main**: 프로덕션 배포

#### Multibranch Pipeline 생성

```
Jenkins > New Item

Name: signbell-multibranch
Type: Multibranch Pipeline

Branch Sources:
- Git
- Repository URL: https://github.com/SynergySign/SignBell-App.git
- Credentials: github-credentials
- Discover branches: All branches

Build Configuration:
- Script Path: Jenkinsfile

Scan Triggers:
- Periodically if not otherwise run: 1 minute

Save
```

#### 배포 프로세스

```bash
# 1. deploy 브랜치에서 테스트
git checkout deploy
git merge feature/new-feature
git push origin deploy
# → Jenkins 자동 빌드 (이미지 태그: deploy-123)

# 2. 테스트 완료 후 main 병합
git checkout main
git merge deploy
git push origin main
# → Jenkins 자동 빌드 (이미지 태그: main-124)
```

### 5.5 Jenkins 자동화 설정 완료 체크리스트

#### ✅ 필수 설정 확인

**1. Jenkins Credentials (3개)**
```
Jenkins > Manage Jenkins > Credentials > System > Global credentials

1. aws-credentials (AWS Credentials)
   - Access Key ID: IAM 사용자 키
   - Secret Access Key: IAM 사용자 시크릿

2. github-credentials (Secret text)
   - Secret: GitHub Personal Access Token

3. kubeconfig (Secret file)
   - File: ~/.kube/config
```

**2. Docker Agent 설정 (3개 템플릿)**
```
Jenkins > Manage Jenkins > Clouds > Docker

Docker Host URI: unix:///var/run/docker.sock

Agent 템플릿:
- fe-agent: node:20-bullseye
- be-agent: gradle:8-jdk17
- ai-agent: python:3.11-slim

모든 템플릿에 Volume 추가:
/var/run/docker.sock:/var/run/docker.sock
```

**3. Pipeline Job 설정**
```
Jenkins > New Item > Pipeline

Name: signbell-pipeline
Type: Pipeline

General:
- GitHub project: ✓
- Project url: https://github.com/SynergySign/SignBell-App

Build Triggers:
- GitHub hook trigger for GITScm polling: ✓

Pipeline:
- Definition: Pipeline script from SCM
- SCM: Git
- Repository URL: https://github.com/SynergySign/SignBell-App.git
- Credentials: github-credentials
- Branches to build:
  - Branch Specifier: */deploy (기본 브랜치)
- Script Path: Jenkinsfile

Save
```

**⚠️ 중요**:
- 현재 Jenkinsfile이 `deploy` 브랜치에 있으므로 `*/deploy`로 설정
- 멀티브랜치 파이프라인을 사용하면 dev, deploy, main 모두 자동 감지

#### 🧪 빌드 테스트

**수동 빌드**:
```
Jenkins > signbell-pipeline > Build Now
```

**자동 빌드 (Git Push)**:
```bash
cd SignBell-App
echo "# Jenkins Test" >> README.md
git add README.md
git commit -m "test: Jenkins 자동 빌드 테스트"
git push origin main

# Jenkins에서 자동으로 빌드 시작 확인
```

**빌드 로그 확인**:
```
Jenkins > signbell-pipeline > #빌드번호 > Console Output
```

#### 🔧 트러블슈팅

**Agent 연결 실패**:
```bash
# Jenkins EC2에서 실행
docker exec -it jenkins docker ps

# Docker 권한 확인
docker exec -it jenkins chmod 666 /var/run/docker.sock
```

**AWS Credentials 오류**:
```bash
# IAM 사용자 확인
aws iam get-user --profile signbell

# Access Key 확인
aws iam list-access-keys --user-name signbell-deployer --profile signbell
```

**Kubeconfig 오류**:
```bash
# kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name signbell-cluster --profile signbell

# kubectl 테스트
kubectl get nodes
```

---

## 6. GitHub 워크플로우 및 브랜치 전략

### 6.1 브랜치 전략 개요

SignBell 프로젝트는 **Git Flow 기반 3단계 브랜치 전략**을 사용합니다.

```
개인 브랜치 (feature/*)
    ↓ PR
  dev (개발 환경)
    ↓ PR
 deploy (스테이징 환경)
    ↓ PR
  main (프로덕션 환경)
```

#### 브랜치별 역할

| 브랜치 | 환경 | 용도 | 자동 배포 | 보호 규칙 |
|--------|------|------|-----------|-----------|
| **feature/*** | 로컬 | 개인 개발 | ❌ | ❌ |
| **dev** | 개발 | 기능 통합 및 테스트 | ✅ | ✅ PR 필수 |
| **deploy** | 스테이징 | 배포 전 최종 검증 | ✅ | ✅ PR 필수 |
| **main** | 프로덕션 | 실제 서비스 운영 | ✅ | ✅ PR + 승인 필수 |

### 6.2 전체 개발 워크플로우

#### 단계별 작업 흐름

```
1. 이슈 생성 (GitHub Issues)
   ↓
2. 개인 브랜치 생성 (feature/issue-번호-기능명)
   ↓
3. 로컬 개발 및 커밋
   ↓
4. dev 브랜치로 PR 생성
   ↓
5. 코드 리뷰 및 병합
   ↓
6. Jenkins 자동 빌드 (dev 환경 배포)
   ↓
7. dev 환경에서 테스트
   ↓
8. deploy 브랜치로 PR 생성
   ↓
9. Jenkins 자동 빌드 (스테이징 배포)
   ↓
10. 스테이징 환경에서 최종 검증
   ↓
11. main 브랜치로 PR 생성
   ↓
12. 승인 후 병합
   ↓
13. Jenkins 자동 빌드 (프로덕션 배포)
```

### 6.3 실전 워크플로우 예시

#### 1단계: 이슈 생성

**GitHub Issues에서 작업 생성**:
```
제목: [Feature] 사용자 프로필 이미지 업로드 기능 추가
라벨: enhancement, backend
담당자: @username

내용:
## 📝 작업 내용
- 사용자가 프로필 이미지를 업로드할 수 있는 API 추가
- S3에 이미지 저장
- 썸네일 자동 생성

## ✅ 체크리스트
- [ ] API 엔드포인트 구현
- [ ] S3 업로드 로직 구현
- [ ] 썸네일 생성 로직 구현
- [ ] 단위 테스트 작성

## 🔗 관련 이슈
#123
```

#### 2단계: 개인 브랜치 생성 및 개발

```bash
# 1. 최신 dev 브랜치 가져오기
git checkout dev
git pull origin dev

# 2. 개인 브랜치 생성 (이슈 번호 포함)
git checkout -b feature/issue-456-profile-image-upload

# 3. 개발 작업
# - 코드 작성
# - 로컬 테스트

# 4. 커밋 (Conventional Commits 규칙)
git add .
git commit -m "feat: 사용자 프로필 이미지 업로드 API 추가

- S3 업로드 서비스 구현
- 썸네일 생성 로직 추가
- 단위 테스트 작성

Resolves #456"

# 5. 원격 저장소에 푸시
git push origin feature/issue-456-profile-image-upload
```

**커밋 메시지 규칙 (Conventional Commits)**:
```
feat: 새로운 기능 추가
fix: 버그 수정
docs: 문서 수정
style: 코드 포맷팅 (기능 변경 없음)
refactor: 코드 리팩토링
test: 테스트 코드 추가/수정
chore: 빌드 설정, 패키지 매니저 설정 등
```

#### 3단계: dev 브랜치로 PR 생성

**GitHub에서 Pull Request 생성**:
```
제목: [Feature] 사용자 프로필 이미지 업로드 기능 추가 (#456)

Base: dev ← Compare: feature/issue-456-profile-image-upload

내용:
## 📝 변경 사항
- 사용자 프로필 이미지 업로드 API 추가 (`POST /api/users/me/profile-image`)
- S3 업로드 서비스 구현
- 썸네일 자동 생성 (200x200)

## 🧪 테스트
- [x] 로컬 테스트 완료
- [x] 단위 테스트 작성
- [ ] dev 환경 테스트 필요

## 📸 스크린샷
(필요시 첨부)

## 🔗 관련 이슈
Closes #456

## ✅ 체크리스트
- [x] 코드 리뷰 준비 완료
- [x] 테스트 코드 작성
- [x] 문서 업데이트 (필요시)
- [x] 커밋 메시지 규칙 준수
```

**PR 라벨 설정**:
- `enhancement`: 새 기능
- `bug`: 버그 수정
- `documentation`: 문서 작업
- `backend`: 백엔드 변경
- `frontend`: 프론트엔드 변경
- `ai`: AI 서버 변경

#### 4단계: 코드 리뷰

**리뷰어 지정 및 리뷰 진행**:
```
리뷰어: @team-lead, @backend-dev

리뷰 코멘트 예시:
- "S3 업로드 실패 시 에러 핸들링이 필요합니다"
- "썸네일 크기를 환경 변수로 관리하는 게 좋을 것 같습니다"
- "LGTM! (Looks Good To Me)"
```

**수정 사항 반영**:
```bash
# 1. 리뷰 코멘트 반영
# - 코드 수정

# 2. 추가 커밋
git add .
git commit -m "fix: S3 업로드 실패 시 에러 핸들링 추가"
git push origin feature/issue-456-profile-image-upload

# PR에 자동으로 반영됨
```

#### 5단계: dev 브랜치 병합 및 자동 배포

**PR 승인 후 병합**:
```
Merge 방식: Squash and merge (권장)
- 여러 커밋을 하나로 합쳐서 히스토리 깔끔하게 유지

병합 후:
1. GitHub Webhook이 Jenkins에 알림
2. Jenkins가 자동으로 빌드 시작
3. dev 환경에 자동 배포
```

**Jenkins 자동 빌드 확인**:
```
Jenkins > signbell-multibranch > dev 브랜치

빌드 로그:
- Frontend 빌드 및 S3 업로드
- Backend 빌드 및 ECR 푸시
- AI Server 빌드 및 ECR 푸시
- EKS 배포 (이미지 태그: dev-123)
```

#### 6단계: dev 환경 테스트

```bash
# dev 환경 접속
# - Frontend: https://dev.signbell.app
# - Backend: https://api-dev.signbell.app
# - AI Server: https://ai-dev.signbell.app

# 기능 테스트
1. 프로필 이미지 업로드 테스트
2. 썸네일 생성 확인
3. 에러 케이스 테스트

# 문제 발견 시
- 추가 수정 후 dev 브랜치에 직접 푸시 또는
- 새로운 PR 생성
```

#### 7단계: deploy 브랜치로 PR 생성 (스테이징 배포)

**dev → deploy PR 생성**:
```
제목: [Release] v1.2.0 - 프로필 이미지 업로드 기능

Base: deploy ← Compare: dev

내용:
## 📦 릴리스 내용
- 사용자 프로필 이미지 업로드 기능 (#456)
- 알림 설정 UI 개선 (#457)
- 로그인 버그 수정 (#458)

## 🧪 테스트 결과
- [x] dev 환경 테스트 완료
- [ ] 스테이징 환경 테스트 필요

## 🚀 배포 계획
- 배포 일시: 2025-11-05 14:00
- 예상 소요 시간: 10분
- 롤백 계획: kubectl rollout undo
```

**병합 후 자동 배포**:
```
1. Jenkins가 deploy 브랜치 빌드
2. 스테이징 환경에 배포 (이미지 태그: deploy-124)
3. 프로덕션과 동일한 환경에서 최종 검증
```

#### 8단계: 스테이징 환경 최종 검증

```bash
# 스테이징 환경 접속
# - Frontend: https://staging.signbell.app
# - Backend: https://api-staging.signbell.app

# 최종 검증 체크리스트
- [ ] 모든 기능 정상 동작
- [ ] 성능 테스트 (응답 시간, 메모리 사용량)
- [ ] 보안 테스트 (인증, 권한)
- [ ] 크로스 브라우저 테스트
- [ ] 모바일 테스트

# 문제 발견 시
- deploy 브랜치에서 핫픽스 또는
- dev로 돌아가서 수정 후 재배포
```

#### 9단계: main 브랜치로 PR 생성 (프로덕션 배포)

**deploy → main PR 생성**:
```
제목: [Production Release] v1.2.0

Base: main ← Compare: deploy

내용:
## 🚀 프로덕션 배포
버전: v1.2.0
배포 일시: 2025-11-05 15:00

## 📦 포함된 기능
- 사용자 프로필 이미지 업로드 기능 (#456)
- 알림 설정 UI 개선 (#457)
- 로그인 버그 수정 (#458)

## ✅ 검증 완료
- [x] dev 환경 테스트
- [x] 스테이징 환경 테스트
- [x] 성능 테스트
- [x] 보안 테스트

## 📊 영향 범위
- 사용자: 프로필 이미지 업로드 가능
- 시스템: S3 스토리지 사용량 증가 예상

## 🔄 롤백 계획
kubectl rollout undo deployment/signbell-backend-deployment

## 👥 승인자
@team-lead, @project-manager
```

**승인 및 병합**:
```
1. 팀 리드 또는 프로젝트 매니저 승인 필수
2. Merge 방식: Create a merge commit (프로덕션은 히스토리 보존)
3. 병합 후 Jenkins 자동 빌드
4. 프로덕션 환경 배포 (이미지 태그: main-125)
```

#### 10단계: 프로덕션 배포 모니터링

```bash
# 배포 상태 확인
kubectl rollout status deployment/signbell-backend-deployment

# Pod 상태 확인
kubectl get pods -l app=signbell-backend

# 실시간 로그 확인
kubectl logs -f deployment/signbell-backend-deployment

# 메트릭 확인
kubectl top pods

# 에러 발생 시 즉시 롤백
kubectl rollout undo deployment/signbell-backend-deployment
```

### 6.4 GitHub 브랜치 보호 규칙 설정

> **중요**: 브랜치 보호 규칙을 설정하면 PR 없이 직접 푸시가 불가능하며, 코드 리뷰 후에만 병합 가능합니다.

#### deploy 브랜치 보호 (스테이징)

```
GitHub Repository > Settings > Branches > Add branch protection rule

Branch name pattern: deploy

규칙:
✅ Require a pull request before merging
  ✅ Require approvals: 1 (코드 리뷰 필수)
✅ Require status checks to pass before merging
  - Jenkins CI (선택사항)
✅ Require conversation resolution before merging
✅ Do not allow bypassing the above settings

Save changes
```

**효과**:
- deploy 브랜치에 직접 푸시 불가
- PR 생성 → 코드 리뷰 → 병합 → Jenkins 자동 배포

#### main 브랜치 보호 (프로덕션)

```
Branch name pattern: main

규칙:
✅ Require a pull request before merging
  ✅ Require approvals: 2 (팀 리드 + 1명, 더 엄격)
  ✅ Dismiss stale pull request approvals when new commits are pushed
✅ Require status checks to pass before merging
  - Jenkins CI (선택사항)
✅ Require conversation resolution before merging
✅ Require linear history
✅ Lock branch (긴급 상황 외 직접 푸시 금지)
✅ Do not allow bypassing the above settings

Save changes
```

**효과**:
- main 브랜치에 직접 푸시 불가
- 2명 이상의 승인 필요
- PR 생성 → 코드 리뷰 → 승인 → 병합 → Jenkins 자동 배포

#### 브랜치 보호 후 워크플로우

```
개인 브랜치 (feature/issue-123)
    ↓ 
    ↓ PR 생성 (deploy ← feature)
    ↓ 코드 리뷰 (1명 승인 필요)
    ↓ 
deploy 브랜치 (보호됨)
    ↓ 
    ↓ PR 병합 → GitHub Webhook
    ↓ 
Jenkins 자동 빌드 🚀
    ↓ 
스테이징 환경 배포
    ↓ 
    ↓ 테스트 완료 후 PR 생성 (main ← deploy)
    ↓ 코드 리뷰 (2명 승인 필요)
    ↓ 
main 브랜치 (보호됨)
    ↓ 
    ↓ PR 병합 → GitHub Webhook
    ↓ 
Jenkins 자동 빌드 🚀
    ↓ 
프로덕션 환경 배포
```

### 6.5 Jenkins 멀티브랜치 파이프라인 설정

#### Jenkinsfile 브랜치별 환경 설정

```groovy
pipeline {
    agent none

    environment {
        AWS_REGION = 'ap-northeast-2'
        ECR_REGISTRY = '801490935747.dkr.ecr.ap-northeast-2.amazonaws.com'
        BACKEND_REPO = 'signbell-backend'
        AI_REPO = 'signbell-ai'
        
        // 브랜치별 환경 설정
        S3_BUCKET = "${env.BRANCH_NAME == 'main' ? 'www.signbell.app' : 
                      env.BRANCH_NAME == 'deploy' ? 'staging.signbell.app' : 
                      'dev.signbell.app'}"
        CF_DISTRIBUTION = "${env.BRANCH_NAME == 'main' ? 'E1BB9LGYVUR99C' : 
                            env.BRANCH_NAME == 'deploy' ? 'E2CC8MHZWVS88D' : 
                            'E3DD9NIAXWT77E'}"
        IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"
    }

    stages {
        stage('Build Frontend') {
            agent { 
                docker {
                    image 'node:20-bullseye'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh """
                    NODE_ENV=${env.BRANCH_NAME == 'main' ? 'production' : 'development'} \
                    VITE_API_URL=${env.BRANCH_NAME == 'main' ? 'https://api.signbell.app' : 
                                   env.BRANCH_NAME == 'deploy' ? 'https://api-staging.signbell.app' : 
                                   'https://api-dev.signbell.app'} \
                    npm run build
                    """
                }
            }
        }

        stage('Deploy Frontend') {
            agent { 
                docker {
                    image 'node:20-bullseye'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('frontend/dist') {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh """
                        apt-get update && apt-get install -y awscli
                        aws s3 sync . s3://${S3_BUCKET} --delete
                        aws cloudfront create-invalidation --distribution-id ${CF_DISTRIBUTION} --paths '/*'
                        """
                    }
                }
            }
        }

        stage('Build Backend') {
            agent { 
                docker {
                    image 'gradle:8-jdk17'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('backend') {
                    sh './gradlew clean build -x test'
                }
            }
        }

        stage('Push Backend Image') {
            agent { 
                docker {
                    image 'gradle:8-jdk17'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('backend') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                            sh """
                            apt-get update && apt-get install -y awscli docker.io
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker build --platform linux/amd64 -t ${BACKEND_REPO}:${IMAGE_TAG} .
                            docker tag ${BACKEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy Backend') {
            agent { 
                docker {
                    image 'gradle:8-jdk17'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    apt-get update && apt-get install -y curl
                    curl -LO "https://dl.k8s.io/release/\$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    mv kubectl /usr/local/bin/
                    
                    export KUBECONFIG=\$KUBECONFIG_FILE
                    kubectl set image deployment/signbell-backend-deployment \
                      signbell-backend=${ECR_REGISTRY}/${BACKEND_REPO}:${IMAGE_TAG} \
                      -n default
                    kubectl rollout status deployment/signbell-backend-deployment -n default --timeout=5m
                    """
                }
            }
        }

        stage('Push AI Image') {
            agent { 
                docker {
                    image 'python:3.11-slim'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                dir('ai-server') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                            sh """
                            apt-get update && apt-get install -y awscli docker.io
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                            docker build --platform linux/amd64 -t ${AI_REPO}:${IMAGE_TAG} .
                            docker tag ${AI_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${AI_REPO}:${IMAGE_TAG}
                            docker push ${ECR_REGISTRY}/${AI_REPO}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy AI Server') {
            agent { 
                docker {
                    image 'python:3.11-slim'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_FILE')]) {
                    sh """
                    apt-get update && apt-get install -y curl
                    curl -LO "https://dl.k8s.io/release/\$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    mv kubectl /usr/local/bin/
                    
                    export KUBECONFIG=\$KUBECONFIG_FILE
                    kubectl set image deployment/signbell-ai-deployment \
                      signbell-ai=${ECR_REGISTRY}/${AI_REPO}:${IMAGE_TAG} \
                      -n default
                    kubectl rollout status deployment/signbell-ai-deployment -n default --timeout=5m
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ ${env.BRANCH_NAME} 브랜치 배포 성공! (이미지 태그: ${IMAGE_TAG})"
        }
        failure {
            echo "❌ ${env.BRANCH_NAME} 브랜치 배포 실패!"
        }
        always {
            cleanWs()
        }
    }
}
```

### 6.6 핫픽스 워크플로우 (긴급 버그 수정)

프로덕션에서 긴급하게 수정이 필요한 경우:

```bash
# 1. main 브랜치에서 핫픽스 브랜치 생성
git checkout main
git pull origin main
git checkout -b hotfix/critical-bug-fix

# 2. 버그 수정
# - 최소한의 변경만 수행

# 3. 커밋 및 푸시
git add .
git commit -m "hotfix: 로그인 실패 버그 긴급 수정"
git push origin hotfix/critical-bug-fix

# 4. main 브랜치로 직접 PR 생성
# - 긴급 상황이므로 빠른 리뷰 및 승인
# - 병합 후 즉시 배포

# 5. 핫픽스를 deploy, dev 브랜치에도 반영
git checkout deploy
git merge main
git push origin deploy

git checkout dev
git merge main
git push origin dev
```

### 6.7 브랜치 전략 요약

#### 일반 개발 흐름
```
feature/* → dev → deploy → main
(개인 개발) (개발 환경) (스테이징) (프로덕션)
```

#### 핫픽스 흐름
```
hotfix/* → main → deploy → dev
(긴급 수정) (프로덕션) (스테이징) (개발 환경)
```

#### 브랜치별 이미지 태그
```
dev 브랜치: dev-123
deploy 브랜치: deploy-124
main 브랜치: main-125
```

---

## 7. 모니터링 및 운영

### 7.1 Pod 상태 확인

```bash
# 전체 Pod 조회
kubectl get pods -A

# 특정 Deployment Pod 조회
kubectl get pods -l app=signbell-backend
kubectl get pods -l app=signbell-ai

# Pod 상세 정보
kubectl describe pod <pod-name>

# Pod 이벤트 확인
kubectl get events --sort-by='.lastTimestamp'
```

### 7.2 실시간 로그 확인

```bash
# Backend 로그
kubectl logs -f deployment/signbell-backend-deployment

# AI Server 로그
kubectl logs -f deployment/signbell-ai-deployment

# 특정 Pod 로그
kubectl logs -f <pod-name>

# 이전 컨테이너 로그 (재시작된 경우)
kubectl logs <pod-name> --previous

# 여러 Pod 로그 동시 확인
kubectl logs -f -l app=signbell-backend --all-containers=true
```

### 7.3 리소스 사용량 모니터링

```bash
# Metrics Server 설치 (최초 1회)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 노드 리소스 사용량
kubectl top nodes

# Pod 리소스 사용량
kubectl top pods

# 특정 Namespace Pod 리소스
kubectl top pods -n default

# 실시간 모니터링 (watch)
watch -n 2 kubectl top pods
```

### 7.4 메모리 및 CPU 상세 확인

```bash
# Pod 리소스 상세 정보
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"

# 컨테이너별 리소스 사용량
kubectl top pod <pod-name> --containers

# OOMKilled 확인 (메모리 부족으로 종료된 Pod)
kubectl get pods | grep OOMKilled
kubectl describe pod <pod-name> | grep -i "oom"

# 리소스 부족 이벤트 확인
kubectl get events --field-selector reason=OOMKilling
```

### 7.5 Deployment 롤아웃 관리

```bash
# 롤아웃 상태 확인
kubectl rollout status deployment/signbell-backend-deployment

# 롤아웃 히스토리
kubectl rollout history deployment/signbell-backend-deployment

# 이전 버전으로 롤백
kubectl rollout undo deployment/signbell-backend-deployment

# 특정 리비전으로 롤백
kubectl rollout undo deployment/signbell-backend-deployment --to-revision=2

# 롤아웃 일시 중지
kubectl rollout pause deployment/signbell-backend-deployment

# 롤아웃 재개
kubectl rollout resume deployment/signbell-backend-deployment
```

### 7.6 ALB 및 Ingress 확인

```bash
# Ingress 상태
kubectl get ingress

# Ingress 상세 정보
kubectl describe ingress signbell-ingress

# ALB 로그 확인 (AWS CLI)
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `k8s-default-signbell`)]' \
  --profile signbell

# ALB 타겟 그룹 상태
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --profile signbell
```

### 7.7 데이터베이스 연결 확인

```bash
# Backend Pod에서 DB 연결 테스트
kubectl exec -it <backend-pod-name> -- bash

# Pod 내부에서 실행
apt-get update && apt-get install -y mariadb-client
mysql -h signbell-db.c9g2mk6gms0l.ap-northeast-2.rds.amazonaws.com \
  -u admin -p signbell

# 연결 성공 시
SHOW DATABASES;
USE signbell;
SHOW TABLES;
```

### 7.8 CloudWatch 로그 (선택사항)

```bash
# EKS 클러스터 로깅 활성화
aws eks update-cluster-config \
  --name signbell-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
  --region ap-northeast-2 \
  --profile signbell

# CloudWatch Logs 확인
aws logs describe-log-groups \
  --log-group-name-prefix /aws/eks/signbell-cluster \
  --profile signbell
```

---

## 8. 트러블슈팅

### 8.1 Pod가 CrashLoopBackOff 상태

**증상**: Pod가 계속 재시작됨

**원인 및 해결**:
```bash
# 1. 로그 확인
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# 2. 이벤트 확인
kubectl describe pod <pod-name>

# 3. 일반적인 원인
# - 환경 변수 누락: Secret/ConfigMap 확인
kubectl get secrets
kubectl get configmaps

# - 메모리 부족: 리소스 제한 확인
kubectl describe pod <pod-name> | grep -A 5 "Limits"

# - DB 연결 실패: RDS 보안 그룹 확인
# - 이미지 Pull 실패: ECR 권한 확인
```

### 8.2 ImagePullBackOff 오류

**증상**: 이미지를 가져올 수 없음

**해결**:
```bash
# 1. ECR 이미지 확인
aws ecr describe-images \
  --repository-name signbell-backend \
  --region ap-northeast-2 \
  --profile signbell

# 2. ECR 권한 확인 (Node IAM Role)
aws iam get-role \
  --role-name signbell-eks-node-role \
  --profile signbell

# 3. 이미지 태그 확인
kubectl describe pod <pod-name> | grep "Image:"

# 4. 수동으로 이미지 Pull 테스트
docker pull 801490935747.dkr.ecr.ap-northeast-2.amazonaws.com/signbell-backend:latest
```

### 8.3 ALB가 생성되지 않음

**증상**: Ingress 적용 후 ALB가 생성되지 않음

**해결**:
```bash
# 1. ALB Controller 로그 확인
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# 2. Ingress 이벤트 확인
kubectl describe ingress signbell-ingress

# 3. ServiceAccount 권한 확인
kubectl describe sa aws-load-balancer-controller -n kube-system

# 4. Ingress 어노테이션 확인
kubectl get ingress signbell-ingress -o yaml
```

### 8.4 502 Bad Gateway 오류

**증상**: ALB에서 502 에러 발생

**해결**:
```bash
# 1. Pod 상태 확인
kubectl get pods -l app=signbell-backend

# 2. Service 엔드포인트 확인
kubectl get endpoints signbell-backend-service

# 3. 타겟 그룹 상태 확인
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --profile signbell

# 4. 보안 그룹 확인
# - ALB → 워커 노드 보안 그룹 규칙 확인
# - NodePort 범위 (30000-32767) 허용 확인

# 5. Readiness Probe 확인
kubectl describe pod <pod-name> | grep -A 10 "Readiness"
```

### 8.5 WebRTC 연결 실패

**증상**: Janus WebRTC 연결이 안 됨

**해결**:
```bash
# 1. UDP 포트 확인
# 워커 노드 보안 그룹에서 UDP 10000-20000, 49152-65535 허용 확인

# 2. AI Server Pod 로그 확인
kubectl logs -f deployment/signbell-ai-deployment

# 3. Janus 설정 확인
kubectl exec -it <ai-pod-name> -- cat /etc/janus/janus.jcfg

# 4. 네트워크 테스트
kubectl exec -it <ai-pod-name> -- nc -zvu <external-ip> 10000
```

### 8.6 데이터베이스 연결 실패

**증상**: Backend에서 DB 연결 오류

**해결**:
```bash
# 1. RDS 엔드포인트 확인
aws rds describe-db-instances \
  --db-instance-identifier signbell-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text \
  --profile signbell

# 2. 보안 그룹 확인
# RDS 보안 그룹에서 EKS 워커 노드 보안 그룹 허용 확인

# 3. Secret 확인
kubectl get secret signbell-secrets -o jsonpath='{.data.db-password}' | base64 -d

# 4. Pod에서 직접 연결 테스트
kubectl exec -it <backend-pod-name> -- bash
apt-get update && apt-get install -y mariadb-client
mysql -h <rds-endpoint> -u admin -p
```

### 8.7 Jenkins 빌드 실패

**증상**: Jenkins Pipeline이 실패함

**해결**:
```bash
# 1. Jenkins 로그 확인
# Jenkins UI > 빌드 번호 > Console Output

# 2. Docker 권한 확인
docker exec -it jenkins docker ps

# 3. AWS Credentials 확인
# Jenkins > Credentials > aws-credentials 확인

# 4. Kubeconfig 확인
docker exec -it jenkins cat /var/jenkins_home/.kube/config

# 5. 에이전트 연결 확인
# Jenkins > Manage Jenkins > Nodes
```

### 8.8 AI Server Pod 메모리 부족 (Pending 상태)

**증상**: AI Server Pod이 Pending 상태로 스케줄링되지 않음

**에러 메시지**:
```
Warning  FailedScheduling  0/2 nodes are available: 2 Insufficient memory.
```

**원인**:
- Rolling Update 시 기존 Pod이 종료되지 않아 새 Pod을 위한 메모리 부족
- 기본 Rolling Update 전략은 `maxSurge: 1`로 새 Pod을 먼저 생성하려고 시도

**해결 방법**:

#### 1단계: 즉시 해결 (기존 Pod 삭제)
```bash
# 기존 Running Pod 확인
kubectl get pods -l app=signbell-ai

# 기존 Pod 삭제 (새 Pod이 자동으로 스케줄링됨)
kubectl delete pod <old-pod-name>

# 새 Pod 상태 확인
kubectl get pods -l app=signbell-ai -w
```

#### 2단계: 근본 해결 (Rolling Update 전략 수정)

**k8s/ai-deployment.yaml 수정**:
```yaml
spec:
  progressDeadlineSeconds: 1800
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0        # 추가 Pod 생성 안 함 (메모리 부족 방지)
      maxUnavailable: 1  # 기존 Pod 먼저 종료
  selector:
    matchLabels:
      app: signbell-ai
```

**설명**:
- `maxSurge: 0`: 새 Pod을 추가로 생성하지 않음
- `maxUnavailable: 1`: 기존 Pod을 먼저 종료하고 메모리 확보 후 새 Pod 생성

**배포 순서**:
```
1. 기존 Pod 종료 (Terminating)
   ↓
2. 메모리 확보
   ↓
3. 새 Pod 생성 (Pending → Running)
   ↓
4. 배포 완료
```

**적용**:
```bash
# 수정된 설정 적용
kubectl apply -f k8s/ai-deployment.yaml

# Deployment 재시작
kubectl rollout restart deployment/signbell-ai-deployment

# 배포 상태 확인
kubectl rollout status deployment/signbell-ai-deployment
```

**⚠️ 주의사항**:
- 이 설정은 **무중단 배포가 아닙니다**
- 기존 Pod 종료 → 새 Pod 시작 사이에 **1-2분 다운타임** 발생
- AI Server는 초기화 시간이 길어서 (5-10분) 어차피 다운타임이 있음
- 메모리 부족 문제를 해결하는 것이 우선

**장기 해결책**:
1. **노드 업그레이드**: 더 큰 인스턴스 타입 사용 (t3.medium → t3.large)
2. **리소스 최적화**: AI 모델 경량화 또는 메모리 요청량 조정
3. **HPA 설정**: 트래픽에 따라 자동 스케일링

#### 3단계: 노드 리소스 확인
```bash
# 노드별 리소스 사용량 확인
kubectl top nodes

# 노드별 할당 가능한 리소스 확인
kubectl describe nodes | grep -A 5 "Allocated resources"

# Pod별 리소스 사용량 확인
kubectl top pods -l app=signbell-ai
```

**예상 출력**:
```
NAME                                      CPU(cores)   MEMORY(bytes)
signbell-ai-deployment-xxx-xxx            250m         1800Mi
```

---

## 9. FAQ 및 예상 질문

### 9.1 아키텍처 관련

**Q1: 왜 EKS를 선택했나요? ECS나 EC2 직접 배포는 어떤가요?**

A: EKS를 선택한 이유:
- **확장성**: Kubernetes의 자동 스케일링 기능으로 트래픽 증가에 유연하게 대응
- **고가용성**: 여러 AZ에 Pod 분산 배포로 장애 대응
- **표준화**: Kubernetes는 업계 표준으로 다른 클라우드로 이전 용이
- **관리 편의성**: Deployment, Service, Ingress 등 선언적 관리

ECS 대비 장점:
- 멀티 클라우드 지원 (AWS 종속성 감소)
- 더 풍부한 생태계 (Helm, Operators 등)

EC2 직접 배포 대비 장점:
- 자동 복구 및 스케일링
- 로드 밸런싱 자동 구성
- 무중단 배포 (Rolling Update)

**Q2: 왜 3개 서비스를 분리했나요?**

A: 마이크로서비스 아키텍처의 장점:
- **독립 배포**: Frontend, Backend, AI 각각 독립적으로 배포 가능
- **기술 스택 자유도**: React, Spring Boot, FastAPI 각각 최적의 기술 선택
- **확장성**: AI 서버만 별도로 스케일 아웃 가능
- **장애 격리**: 한 서비스 장애가 다른 서비스에 영향 최소화

**Q3: ALB를 사용하는 이유는?**

A: Application Load Balancer 장점:
- **경로 기반 라우팅**: `/api/*` → Backend, `/ws/*` → AI Server
- **SSL/TLS 종료**: ACM 인증서로 HTTPS 자동 처리
- **Health Check**: 비정상 Pod 자동 제외
- **WebSocket 지원**: AI Server의 실시간 통신 지원

### 9.2 배포 및 CI/CD 관련

**Q4: Jenkins를 선택한 이유는? GitHub Actions는 어떤가요?**

A: Jenkins 선택 이유:
- **비용**: GitHub Actions는 분당 과금, Jenkins는 EC2 비용만
- **유연성**: 복잡한 빌드 파이프라인 구성 가능
- **플러그인**: AWS, Kubernetes 통합 플러그인 풍부
- **학습 목적**: 실무에서 많이 사용하는 도구 경험

GitHub Actions 대비 단점:
- 초기 설정 복잡
- 서버 관리 필요

**Q5: 무중단 배포는 어떻게 구현했나요?**

A: Kubernetes Rolling Update 전략 (서비스별 차이):

**Backend (무중단 배포)**:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 추가로 생성할 Pod 수
    maxUnavailable: 0  # 동시에 종료할 Pod 수
```

배포 과정:
1. 새 버전 Pod 1개 생성
2. Readiness Probe 통과 확인
3. 기존 Pod 1개 종료
4. 모든 Pod 교체 완료까지 반복

**AI Server (메모리 제약으로 순차 배포)**:
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0        # 추가 Pod 생성 안 함
    maxUnavailable: 1  # 기존 Pod 먼저 종료
```

배포 과정:
1. 기존 Pod 종료 (메모리 확보)
2. 새 버전 Pod 생성
3. 1-2분 다운타임 발생 (AI 모델 로딩 시간 고려)

**차이점**:
- Backend: 충분한 메모리로 무중단 배포 가능
- AI Server: 메모리 부족으로 순차 배포 (다운타임 있음)

**Q6: 배포 실패 시 롤백은 어떻게 하나요?**

A: 자동 롤백 및 수동 롤백:
```bash
# 자동 롤백 (Readiness Probe 실패 시)
# Kubernetes가 자동으로 이전 버전 유지

# 수동 롤백
kubectl rollout undo deployment/signbell-backend-deployment

# 특정 버전으로 롤백
kubectl rollout undo deployment/signbell-backend-deployment --to-revision=2
```

### 9.3 보안 관련

**Q7: 민감 정보는 어떻게 관리하나요?**

A: Kubernetes Secret 사용:
- DB 비밀번호, JWT Secret, API Key 등을 Secret으로 관리
- Base64 인코딩 (암호화 아님, 주의!)
- 프로덕션에서는 AWS Secrets Manager 또는 HashiCorp Vault 권장

```bash
# Secret 생성
kubectl create secret generic signbell-secrets \
  --from-literal=db-password='...'

# Pod에서 환경 변수로 주입
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: signbell-secrets
      key: db-password
```

**Q8: HTTPS는 어떻게 적용했나요?**

A: 2가지 방식:
1. **ALB (Backend/AI)**: ACM 인증서 + ALB Listener
2. **CloudFront (Frontend)**: ACM 인증서 (us-east-1) + CloudFront

```yaml
# Ingress에서 SSL 설정
annotations:
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
```

### 9.4 모니터링 및 운영 관련

**Q9: 로그는 어떻게 확인하나요?**

A: 3가지 방법:
```bash
# 1. kubectl로 실시간 로그
kubectl logs -f deployment/signbell-backend-deployment

# 2. CloudWatch Logs (EKS 클러스터 로깅 활성화 시)
aws logs tail /aws/eks/signbell-cluster/cluster --follow

# 3. 프로덕션에서는 ELK Stack 또는 Datadog 권장
```

**Q10: 리소스 사용량은 어떻게 모니터링하나요?**

A: Metrics Server 사용:
```bash
# 설치
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 노드 리소스
kubectl top nodes

# Pod 리소스
kubectl top pods

# 실시간 모니터링
watch -n 2 kubectl top pods
```

프로덕션에서는 Prometheus + Grafana 권장

**Q11: 장애 발생 시 어떻게 대응하나요?**

A: 장애 대응 프로세스:
1. **알림 수신**: CloudWatch Alarm 또는 Slack 알림
2. **로그 확인**: `kubectl logs -f <pod-name>`
3. **Pod 상태 확인**: `kubectl get pods`, `kubectl describe pod`
4. **롤백**: `kubectl rollout undo deployment/...`
5. **원인 분석**: 로그, 이벤트, 메트릭 분석
6. **재배포**: 수정 후 재배포

### 9.5 비용 관련

**Q12: 월 예상 비용은 얼마인가요?**

A: 주요 비용 항목 (ap-northeast-2 기준):
- **EKS Control Plane**: $73/월
- **EC2 (t3.small x2)**: ~$30/월
- **RDS (db.t3.micro)**: ~$15/월
- **ALB**: ~$18/월
- **NAT Gateway**: ~$35/월
- **S3 + CloudFront**: ~$5/월
- **ECR**: ~$1/월

**총 예상 비용**: **약 $177/월 (~23만원)**

비용 절감 방법:
- Spot Instance 사용 (최대 90% 절감)
- NAT Gateway 대신 NAT Instance
- Reserved Instance (1년 약정 시 40% 절감)

**Q13: 프리 티어로 운영 가능한가요?**

A: 불가능합니다. 이유:
- EKS Control Plane은 프리 티어 미포함 ($73/월)
- ALB는 프리 티어 미포함 (~$18/월)
- NAT Gateway는 프리 티어 미포함 (~$35/월)

프리 티어 활용 가능 항목:
- EC2 t3.micro (750시간/월)
- RDS db.t3.micro (750시간/월)
- S3 (5GB)
- CloudFront (1TB 전송)

### 9.6 GitHub 워크플로우 관련

**Q14: 개인 브랜치에서 개발 후 어떻게 배포까지 진행하나요?**

A: 단계별 워크플로우:
```
1. GitHub Issues에 작업 등록 (#456)
2. 개인 브랜치 생성 (feature/issue-456-feature-name)
3. 로컬 개발 및 커밋
4. dev 브랜치로 PR 생성
5. 코드 리뷰 및 병합
6. Jenkins 자동 빌드 (dev 환경 배포)
7. dev 환경 테스트
8. deploy 브랜치로 PR 생성
9. Jenkins 자동 빌드 (스테이징 배포)
10. 스테이징 환경 최종 검증
11. main 브랜치로 PR 생성
12. 승인 후 병합
13. Jenkins 자동 빌드 (프로덕션 배포)
```

자세한 내용은 [GitHub 워크플로우 가이드](./GITHUB_WORKFLOW_GUIDE.md) 참고

**Q15: PR 없이 직접 dev 브랜치에 푸시하면 안 되나요?**

A: 가능하지만 권장하지 않습니다.

PR을 사용하는 이유:
- **코드 리뷰**: 버그를 사전에 발견
- **히스토리 관리**: 어떤 변경이 언제 왜 이루어졌는지 추적
- **CI/CD 통합**: PR 단계에서 자동 테스트 실행
- **팀 협업**: 변경 사항을 팀원에게 공유

예외 상황:
- 긴급 핫픽스 (하지만 이것도 PR 권장)
- 문서 수정 (README, 주석 등)

**Q16: 여러 기능을 동시에 개발 중인데 어떻게 관리하나요?**

A: 각 기능별로 별도 브랜치 생성:
```bash
# 기능 A 개발
git checkout -b feature/issue-100-feature-a

# 기능 B 개발 (동시 진행)
git checkout dev
git checkout -b feature/issue-101-feature-b

# 각각 독립적으로 PR 생성 및 병합
# dev 브랜치에서 통합 테스트
```

충돌 방지 방법:
- 자주 dev 브랜치를 pull 받아서 최신 상태 유지
- 같은 파일을 수정하는 경우 팀원과 사전 조율
- 작은 단위로 자주 병합 (Big Bang 방지)

**Q17: deploy 브랜치는 왜 필요한가요? dev에서 바로 main으로 가면 안 되나요?**

A: deploy 브랜치의 역할:

**deploy 브랜치가 있는 경우**:
```
dev (개발 환경)
- 빠르게 변경, 버그 있을 수 있음
- 개발자들이 자유롭게 테스트

deploy (스테이징 환경)
- 프로덕션과 동일한 환경
- 최종 검증 (성능, 보안, 통합 테스트)
- 배포 전 마지막 관문

main (프로덕션 환경)
- 항상 안정적
- 실제 사용자에게 서비스
```

**deploy 브랜치가 없는 경우**:
```
dev → main
- dev에서 발견 못한 버그가 프로덕션에 바로 배포
- 위험도 높음
```

프로젝트 규모가 작거나 초기 단계라면 dev → main도 가능하지만, 안정성을 위해 3단계 전략 권장

**Q18: Jenkins가 자동으로 빌드를 시작하는 원리는?**

A: GitHub Webhook 메커니즘:
```
1. 개발자가 코드 푸시
   git push origin dev

2. GitHub가 Webhook 이벤트 발생
   POST http://<Jenkins-IP>:8080/github-webhook/
   {
     "ref": "refs/heads/dev",
     "commits": [...],
     "repository": {...}
   }

3. Jenkins가 Webhook 수신
   - 어떤 브랜치에 푸시되었는지 확인
   - 해당 브랜치의 Jenkinsfile 실행

4. Jenkinsfile 실행
   - Frontend 빌드
   - Backend 빌드
   - AI Server 빌드
   - 각각 배포

5. 빌드 결과 알림
   - GitHub PR에 빌드 상태 표시
   - Slack 알림 (설정 시)
```

Webhook 설정 확인:
```
GitHub Repository > Settings > Webhooks
Payload URL: http://<Jenkins-IP>:8080/github-webhook/
Content type: application/json
Events: Just the push event
```

### 9.7 성능 및 확장성 관련

**Q19: 트래픽이 증가하면 어떻게 대응하나요?**

A: 자동 스케일링 설정:
```yaml
# HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: signbell-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: signbell-backend-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Q20: 데이터베이스 성능 최적화는?**

A: RDS 최적화 방법:
- **Read Replica**: 읽기 부하 분산
- **Connection Pooling**: HikariCP 설정 최적화
- **인덱스**: 자주 조회하는 컬럼에 인덱스 추가
- **쿼리 최적화**: N+1 문제 해결, Lazy Loading
- **캐싱**: Redis 도입 (세션, 자주 조회하는 데이터)

---

## 9. 참고 자료

### 9.1 공식 문서
- [AWS EKS 문서](https://docs.aws.amazon.com/eks/)
- [Kubernetes 문서](https://kubernetes.io/docs/)
- [Jenkins 문서](https://www.jenkins.io/doc/)
- [Docker 문서](https://docs.docker.com/)

### 9.2 유용한 명령어 모음

```bash
# EKS 클러스터 정보
kubectl cluster-info

# 모든 리소스 조회
kubectl get all -A

# 특정 Namespace 리소스
kubectl get all -n default

# 리소스 삭제
kubectl delete deployment <name>
kubectl delete service <name>
kubectl delete ingress <name>

# 강제 삭제
kubectl delete pod <name> --force --grace-period=0

# 설정 파일 검증
kubectl apply --dry-run=client -f deployment.yaml

# 리소스 편집
kubectl edit deployment <name>

# 포트 포워딩 (로컬 테스트)
kubectl port-forward service/signbell-backend-service 8080:8080

# 컨테이너 접속
kubectl exec -it <pod-name> -- /bin/bash

# 파일 복사
kubectl cp <pod-name>:/path/to/file ./local-file
```

---

## 10. 마무리

이 가이드는 SignBell 프로젝트의 전체 배포 과정을 다룹니다.

**핵심 포인트**:
1. ✅ **인프라 자동화**: eksctl, Helm으로 인프라 코드화
2. ✅ **CI/CD 파이프라인**: Jenkins로 3개 서비스 자동 배포
3. ✅ **모니터링**: kubectl, Metrics Server로 실시간 모니터링
4. ✅ **무중단 배포**: Rolling Update로 서비스 중단 없이 배포
5. ✅ **보안**: Secret, IAM Role, 보안 그룹으로 보안 강화

**다음 단계**:
- Prometheus + Grafana 모니터링 구축
- ELK Stack 로그 수집
- ArgoCD GitOps 도입
- Istio Service Mesh 적용

---

## 11. 변경 이력

| 버전     | 날짜         | 변경 내용                        | 작성자 |
|--------|------------|------------------------------|-----|
| v1.0.0 | 2025.11.03 | 배포 가이드 통합 문서 작성 | 신동준 |
