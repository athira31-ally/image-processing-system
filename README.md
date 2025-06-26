# Image Processing System

Image processing API with FastAPI and Celery, deployed on AWS using ECS Fargate, SQS, and S3.

## What This Does

Upload images and get them automatically processed (resized, compressed, format conversion) in the background while you get an immediate response.

## Architecture

```
FastAPI API ────► Amazon SQS ────► Celery Worker
(ECS Fargate)    (Message Queue)    (ECS Fargate)
     │                                   │
     │                                   │
     ▼                                   ▼
   Amazon S3 ◄─────────────────── Image Processing
 (File Storage)
```

## Live Demo

http://image-processor-alb-1146437516.ap-south-1.elb.amazonaws.com

```bash
# Quick test
curl -X POST "http://image-processor-alb-1146437516.ap-south-1.elb.amazonaws.com/upload-image/" -F "file=@your-image.jpg"
```

## Setup Instructions



### Step 1: Create AWS IAM User 

1. **Log in to AWS Console:**
   - Go to [console.aws.amazon.com](https://console.aws.amazon.com)
   - Sign in with your root account

2. **Navigate to IAM:**
   - Search for "IAM" in the services search bar
   - Click on "IAM" service

3. **Create New User:**
   - Click "Users" in left sidebar
   - Click "Create user"
   - Enter username: `image-processor-admin`
   - Check "Provide user access to the AWS Management Console"
   - Choose "I want to create an IAM user"
   - Set console password or use auto-generated
   - Uncheck "User must create a new password at next sign-in"

4. **Attach Policies:**
   - Click "Next: Permissions"
   - Click "Attach policies directly"
   - Search and select these policies:
     - `AmazonECS_FullAccess`
     - `AmazonEC2ContainerRegistryFullAccess`
     - `AmazonS3FullAccess`
     - `AmazonSQSFullAccess`
     - `IAMFullAccess`
     - `ElasticLoadBalancingFullAccess`
   - Click "Next: Tags" (skip tags)
   - Click "Next: Review"
   - Click "Create user"

5. **Create Access Keys:**
   - Click on the created user
   - Go to "Security credentials" tab
   - Click "Create access key"
   - Choose "Command Line Interface (CLI)"
   - Check the confirmation checkbox
   - Click "Next"
   - Add description: "Image Processing CLI Access"
   - Click "Create access key"
   - **IMPORTANT**: Copy and save both:
     - Access Key ID
     - Secret Access Key
   - Click "Done"

### Step 2: Install and Configure AWS CLI

1. **Install AWS CLI:**

   **On Windows:**
   - Download AWS CLI installer from [AWS CLI Windows](https://awscli.amazonaws.com/AWSCLIV2.msi)
   - Run the installer

   **On macOS:**
   ```bash
   curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
   sudo installer -pkg AWSCLIV2.pkg -target /
   ```

   

2. **Configure AWS CLI:**
   ```bash
   aws configure
   ```
   Enter when prompted:
   - **AWS Access Key ID**: (from Step 1)
   - **AWS Secret Access Key**: (from Step 1)
   - **Default region name**: `ap-south-1`
   - **Default output format**: `json`

3. **Verify Configuration:**
   ```bash
   aws sts get-caller-identity
   ```

### Step 3: Create Required AWS Resources

1. **Create S3 Bucket in terminal or can create manually:**
   ```bash
   # Replace 'your-name' with something unique
   aws s3 mb s3://image-processor-your-name-123 --region ap-south-1
   ```

2. **Create SQS Queue in terminal or can create manually:**
   ```bash
   aws sqs create-queue --queue-name image-processor-queue --region ap-south-1
   ```

3. **Get Your Configuration Values:**
   ```bash
   echo "Account ID: $(aws sts get-caller-identity --query Account --output text)"
   echo "SQS URL: $(aws sqs get-queue-url --queue-name image-processor-queue --region ap-south-1 --query QueueUrl --output text)"
   echo "Your bucket: image-processor-your-name-123"
   ```

### Step 4: Clone and Setup Application

1. **Clone Repository:**
   ```bash
   git clone https://github.com/athira31-ally/image-processing-system.git
   cd image-processing-system
   cp .env.example .env
   ```

2. **Configure Environment:**
   ```bash
   nano .env  # or use any text editor
   ```

   Update `.env` with your values:
   ```env
   AWS_ACCESS_KEY_ID=your_access_key_here
   AWS_SECRET_ACCESS_KEY=your_secret_key_here
   AWS_DEFAULT_REGION=ap-south-1
   CELERY_BROKER_URL=https://sqs.ap-south-1.amazonaws.com/YOUR_ACCOUNT_ID/image-processor-queue
   CELERY_RESULT_BACKEND=s3://image-processor-your-name-123/celery-results/
   S3_BUCKET_NAME=image-processor-your-name-123
   ```

### Step 5: Test Locally

1. **Start the Application:**
   ```bash
   docker-compose up --build
   ```

2. **Test It Works:**
   ```bash
   # Test API health
   curl http://localhost:8000/health
   
   # Download test image
   curl -o test.jpg "https://picsum.photos/800/600"
   
   # Upload image
   curl -X POST "http://localhost:8000/upload-image/" -F "file=@test.jpg"
   
   # Check processing status (replace YOUR_TASK_ID with actual ID from response)
   curl "http://localhost:8000/status/YOUR_TASK_ID"
   ```

## Deployment to AWS

### Step 1: Create Container Repositories

```bash
aws ecr create-repository --repository-name image-processor-api --region ap-south-1
aws ecr create-repository --repository-name image-processor-worker --region ap-south-1
```

### Step 2: Build and Push Docker Images

```bash
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Login to AWS container registry
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com

# Build for AWS
docker build --platform linux/amd64 -t $ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/image-processor-api:latest .
docker build --platform linux/amd64 -f Dockerfile.worker -t $ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/image-processor-worker:latest .

# Push to AWS
docker push $ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/image-processor-api:latest
docker push $ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/image-processor-worker:latest
```

### Step 3: Update ECS Services

```bash
# Update ECS services (assumes cluster and services already exist)
aws ecs update-service --cluster image-processor-cluster --service api-service --force-new-deployment
aws ecs update-service --cluster image-processor-cluster --service worker-service --force-new-deployment
```

### Step 4: Monitor Deployment

```bash
aws ecs describe-services --cluster image-processor-cluster --services api-service worker-service --query 'services[*].[serviceName,runningCount,desiredCount]' --output table
```






