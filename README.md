# E-commerce Application Deployment Guide

## 1. Initial Setup

### 1.1 Clone the Repository
```bash
# Clone the repository
git clone <your-repository-url>
cd ecommerce-app

# Create required directories if not present
mkdir -p backend/utils
touch backend/utils/__init__.py
touch backend/utils/password_utils.py
```

### 1.2 Update Database Schema
```bash
# Ensure schema.sql includes is_admin column in users table
ALTER TABLE users ADD COLUMN is_admin BOOLEAN DEFAULT FALSE;
```

## 2. Local Development Setup

### 2.1 Install Required Tools
```bash
# Install Docker
sudo apt update
sudo apt install docker.io docker-compose -y

# Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### 2.2 Build and Run Locally
```bash
# Navigate to docker directory
cd docker

# Build and start services
docker compose up --build -d

# Check logs
docker compose logs -f

# Verify containers are running
docker ps
```

## 3. Docker Image Management

### 3.1 Build Images
```bash
# Build backend image
docker build -t ecommerce-app-backend:latest -f backend/Dockerfile.backend ./backend
```

### 3.2 Push to DockerHub
```bash
# Login to DockerHub
docker login

# Tag images
docker tag ecommerce-app-backend:latest yourusername/ecommerce-app-backend:v1

# Push images
docker push yourusername/ecommerce-app-backend:v1
```

### 3.3 Push to AWS ECR
```bash
# Configure AWS CLI
aws configure

# Create ECR repository
aws ecr create-repository --repository-name ecommerce-app

# Login to ECR
aws ecr get-login-password --region your-region | \
  docker login --username AWS --password-stdin \
  your-account-id.dkr.ecr.your-region.amazonaws.com

# Tag for ECR
docker tag ecommerce-app-backend:latest \
  your-account-id.dkr.ecr.your-region.amazonaws.com/ecommerce-app:backend-v1

# Push to ECR
docker push your-account-id.dkr.ecr.your-region.amazonaws.com/ecommerce-app:backend-v1
```

## 4. Docker Compose Commands Reference

### 4.1 Basic Commands
```bash
# Start services
docker compose -f docker/docker-compose.yaml up -d

# Stop services
docker compose -f docker/docker-compose.yaml down

# View logs
docker compose -f docker/docker-compose.yaml logs -f

# Rebuild and start
docker compose -f docker/docker-compose.yaml up --build -d

# Stop and remove volumes
docker compose -f docker/docker-compose.yaml down -v
```

### 4.2 Container Management
```bash
# List containers
docker ps

# Enter container shell
docker exec -it docker-backend-1 /bin/bash

# View container logs
docker logs -f docker-backend-1
```

## 5. Testing the Application

### 5.1 API Testing
```bash
# Test backend health
curl http://localhost:5000/api/health

# Login as admin
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "subbu", "password": "admin1234"}'
```

## 6. Next Steps

### 6.1 Prepare for Kubernetes Deployment
1. Set up Kubernetes cluster (EKS)
2. Create Helm charts
3. Configure Jenkins pipeline

### 6.2 Required for K8s Setup
1. AWS CLI configured with proper permissions
2. kubectl installed and configured
3. Helm installed
4. Jenkins server with required plugins

## 7. Common Issues and Solutions

### 7.1 Docker Issues
- Permission denied: Add user to docker group
- Port already in use: Change port mapping or stop conflicting service
- Container not starting: Check logs with `docker logs`

### 7.2 Database Issues
- Connection refused: Check if database container is healthy
- Authentication failed: Verify environment variables
- Data persistence: Ensure volumes are properly configured

## 8. Security Best Practices

1. Use environment variables for sensitive data
2. Never commit .env files to repository
3. Regularly update dependencies
4. Use specific image tags instead of 'latest'
5. Implement proper access controls
6. Regular security updates

## 9. Maintenance

1. Regular backups of database
2. Log rotation
3. Monitor resource usage
4. Keep dependencies updated
5. Regular security patches
