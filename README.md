# Kubernetes Deployment Guide with Helm and Jenkins Pipeline

## 1. Helm Chart Structure

```
ecommerce-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── configmaps/
    │   └── backend-config.yaml
    ├── secrets/
    │   ├── backend-secret.yaml
    │   └── mysql-secret.yaml
    ├── deployments/
    │   ├── backend-deployment.yaml
    │   └── mysql-deployment.yaml
    ├── services/
    │   ├── backend-service.yaml
    │   └── mysql-service.yaml
    └── storage/
        ├── mysql-pv.yaml
        └── mysql-pvc.yaml
```

## 2. Helm Chart Configuration Files

### 2.1 Chart.yaml
```yaml
apiVersion: v2
name: ecommerce-app
description: E-commerce application with Python backend and MySQL
version: 0.1.0
type: application
```

### 2.2 values.yaml
```yaml
backend:
  name: ecommerce-backend
  image:
    repository: your-account-id.dkr.ecr.your-region.amazonaws.com/ecommerce-app
    tag: backend-v1
  replicas: 2
  service:
    type: LoadBalancer
    port: 5000
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

mysql:
  name: ecommerce-db
  image:
    repository: mysql
    tag: "8.0"
  service:
    port: 3306
  storage:
    size: 10Gi
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 512Mi

config:
  flaskDebug: "true"
  sessionType: "filesystem"
  sessionFileDir: "/tmp/flask_sessions"

secrets:
  mysqlRootPassword: admin@1234
  mysqlUser: subbu
  mysqlPassword: admin@1234
  mysqlDatabase: ecommerce
  flaskSecretKey: "ecommerce_app_secret_key_2024_secure_shopping"
```

## 3. Kubernetes Templates

### 3.1 ConfigMap (templates/configmaps/backend-config.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-backend-config
data:
  FLASK_APP: "app.py"
  FLASK_RUN_HOST: "0.0.0.0"
  FLASK_RUN_PORT: "5000"
  FLASK_DEBUG: {{ .Values.config.flaskDebug | quote }}
  SESSION_TYPE: {{ .Values.config.sessionType | quote }}
  SESSION_FILE_DIR: {{ .Values.config.sessionFileDir | quote }}
```

### 3.2 Secrets (templates/secrets/backend-secret.yaml)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-backend-secret
type: Opaque
data:
  DB_USER: {{ .Values.secrets.mysqlUser | b64enc }}
  DB_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc }}
  DB_NAME: {{ .Values.secrets.mysqlDatabase | b64enc }}
  SECRET_KEY: {{ .Values.secrets.flaskSecretKey | b64enc }}
```

### 3.3 MySQL Secret (templates/secrets/mysql-secret.yaml)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ .Release.Name }}-mysql-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: {{ .Values.secrets.mysqlRootPassword | b64enc }}
  MYSQL_USER: {{ .Values.secrets.mysqlUser | b64enc }}
  MYSQL_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc }}
  MYSQL_DATABASE: {{ .Values.secrets.mysqlDatabase | b64enc }}
```

### 3.4 MySQL PV (templates/storage/mysql-pv.yaml)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: {{ .Release.Name }}-mysql-pv
spec:
  capacity:
    storage: {{ .Values.mysql.storage.size }}
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  awsElasticBlockStore:
    volumeID: vol-xxxxx # Replace with your EBS volume ID
    fsType: ext4
```

### 3.5 MySQL PVC (templates/storage/mysql-pvc.yaml)
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ .Release.Name }}-mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.mysql.storage.size }}
  storageClassName: ebs-sc
```

### 3.6 MySQL Deployment (templates/deployments/mysql-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: {{ .Values.mysql.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.mysql.name }}
    spec:
      containers:
        - name: mysql
          image: "{{ .Values.mysql.image.repository }}:{{ .Values.mysql.image.tag }}"
          ports:
            - containerPort: 3306
          envFrom:
            - secretRef:
                name: {{ .Release.Name }}-mysql-secret
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
          resources:
            {{- toYaml .Values.mysql.resources | nindent 12 }}
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: {{ .Release.Name }}-mysql-pvc
```

### 3.7 MySQL Service (templates/services/mysql-service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-mysql
spec:
  selector:
    app: {{ .Values.mysql.name }}
  ports:
    - port: {{ .Values.mysql.service.port }}
      targetPort: 3306
  clusterIP: None
```

### 3.8 Backend Deployment (templates/deployments/backend-deployment.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
spec:
  replicas: {{ .Values.backend.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.backend.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.backend.name }}
    spec:
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-backend-config
            - secretRef:
                name: {{ .Release.Name }}-backend-secret
          env:
            - name: DB_HOST
              value: {{ .Release.Name }}-mysql
            - name: DB_PORT
              value: "3306"
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
```

### 3.9 Backend Service (templates/services/backend-service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-backend
spec:
  type: {{ .Values.backend.service.type }}
  selector:
    app: {{ .Values.backend.name }}
  ports:
    - port: {{ .Values.backend.service.port }}
      targetPort: 5000
```

## 4. Jenkins Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any
    
    environment {
        AWS_ACCOUNT_ID = 'your-account-id'
        AWS_DEFAULT_REGION = 'your-region'
        IMAGE_REPO_NAME = 'ecommerce-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        HELM_RELEASE_NAME = 'ecommerce'
        HELM_CHART_PATH = './helm/ecommerce-app'
        KUBECONFIG = credentials('eks-kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${REPOSITORY_URI}:backend-${IMAGE_TAG}", "-f backend/Dockerfile.backend ./backend")
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    sh """
                        aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}
                        docker push ${REPOSITORY_URI}:backend-${IMAGE_TAG}
                    """
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=\${KUBECONFIG}
                        
                        # Update Helm dependencies
                        helm dependency update ${HELM_CHART_PATH}
                        
                        # Deploy/Upgrade Helm chart
                        helm upgrade --install ${HELM_RELEASE_NAME} ${HELM_CHART_PATH} \
                            --namespace ecommerce \
                            --create-namespace \
                            --set backend.image.repository=${REPOSITORY_URI} \
                            --set backend.image.tag=backend-${IMAGE_TAG} \
                            --wait --timeout 10m
                            
                        # Verify deployment
                        kubectl rollout status deployment/${HELM_RELEASE_NAME}-backend -n ecommerce
                    """
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    sh """
                        # Wait for services to be ready
                        sleep 30
                        
                        # Run backend tests
                        kubectl exec -n ecommerce \
                            \$(kubectl get pod -n ecommerce -l app=ecommerce-backend -o jsonpath='{.items[0].metadata.name}') \
                            -- python -m pytest tests/
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
        always {
            // Clean up Docker images
            sh "docker rmi ${REPOSITORY_URI}:backend-${IMAGE_TAG}"
        }
    }
}
```

## 5. Deployment Steps

1. Set up EKS cluster:
```bash
eksctl create cluster --name ecommerce-cluster \
    --region your-region \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 1 \
    --nodes-max 4
```

2. Create storage class for EBS:
```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
EOF
```

3. Install Helm chart:
```bash
# Add required labels to namespace
kubectl create namespace ecommerce
kubectl label namespace ecommerce name=ecommerce

# Install chart
helm upgrade --install ecommerce ./helm/ecommerce-app \
    --namespace ecommerce \
    --values ./helm/ecommerce-app/values.yaml
```

4. Verify deployment:
```bash
# Check pods
kubectl get pods -n ecommerce

# Check services
kubectl get svc -n ecommerce

# Check logs
kubectl logs -f -l app=ecommerce-backend -n ecommerce
```

## 6. Common Issues and Solutions

1. EBS Volume Creation:
   - Ensure proper IAM roles for EBS volume creation
   - Check storage class configuration

2. ECR Authentication:
   - Verify AWS credentials
   - Check ECR repository permissions

3. Database Connection:
   - Verify secrets are properly created
   - Check service DNS resolution
   - Validate database initialization

4. Jenkins Pipeline:
   - Configure AWS credentials in Jenkins
   - Set up proper KUBECONFIG
   - Install required Jenkins plugins

Would you like me to:
1. Add more details to any section?
2. Include monitoring and logging configuration?
3. Add scaling and backup configurations?
