# image-processing-system
Image processing API with FastAPI and Celery

# Image Processing Service
An image processing service built with FastAPI and Celery, deployed on AWS using ECS Fargate, SQS, and S3.

# Architecture flow
       
   FastAPI API   ────------   Amazon SQS               ────  Celery Worker  
   (ECS Fargate)             (Message Queue, Broker)         (ECS Fargate)  
       
         │                                                       │
         │                                                       │
         ▼                                                       ▼
                           
   Amazon S3     ◄───────────────────────────------     Image         
   (File Storage)                                      Processing    
                           
**##Create IAM User (Security Best Practice)**

1. Log in to AWS Console:
       Go to console.aws.amazon.com
       Sign in with your root account
2. Navigate to IAM:
       Search for "IAM" in the services search bar
       Click on "IAM" service
3. Create New User:
       Click "Users" in left sidebar
       Click "Create user"
       Enter username: image-processor-admin
       Check "Provide user access to the AWS Management Console"
       Choose "I want to create an IAM user"
       Set console password or use auto-generated
       Uncheck "User must create a new password at next sign-in"
4. Attach Policies:
       Click "Next: Permissions"
       Click "Attach policies directly"
       Search and select these policies:

       AmazonECS_FullAccess
       AmazonEC2ContainerRegistryFullAccess
       AmazonS3FullAccess
       AmazonSQSFullAccess
       IAMFullAccess
       ElasticLoadBalancingFullAccess
       Click "Next: Tags" (skip tags)
       Click "Next: Review"
       Click "Create user"
5. Create Access Keys:
       Click on the created user
       Go to "Security credentials" tab
       Click "Create access key"
       Choose "Command Line Interface (CLI)"
       Check the confirmation checkbox
       Click "Next"
       Add description: "Image Processing CLI Access"
       Click "Create access key"
       IMPORTANT: Copy and save both:
                     Access Key ID
                     Secret Access Key
       Click "Done"
**Install and Configure AWS CLI**
Install AWS CLI:
       On Windows:
       Download AWS CLI installer from AWS CLI Windows
       Run the installer
On macOS:
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

Configure AWS CLI:  aws configure
              AWS Access Key ID: 
              AWS Secret Access Key:
              Default region name: ap-south-1 
              Default output format: json
aws sts get-caller-identity

**We can either create an S3 bucket manually and have the name, or we can create it  using CLI **

# Create a storage bucket (replace 'your-name' with something unique)
aws s3 mb s3://image-processor-your-name-123 --region ap-south-1

# Create a message queue either manually or through the terminal
aws sqs create-queue --queue-name image-processor-queue --region ap-south-1


**##Clone and Setup**
```bash
git clone https://github.com/athira31-ally/image-processing-system.git
cd image-processing-system
cp .env.example .env
```
### Adding Your AWS Keys to .env
```bash
nano .env
```
Replace with your real AWS credentials:
```
AWS_ACCESS_KEY_ID=AKIA...your_key_here
AWS_SECRET_ACCESS_KEY=...your_secret_here
AWS_DEFAULT_REGION=ap-south-1
CELERY_BROKER_URL=sqs://https://sqs.ap-south-1.amazonaws.com/534211282949/image-processor-queue
CELERY_RESULT_BACKEND=s3://image-processor-bucket-534211282949/celery-results/

S3_BUCKET_NAME=your-image-processing-bucket-1750868832

echo "Account ID: $(aws sts get-caller-identity --query Account --output text)"
echo "SQS URL: $(aws sqs get-queue-url --queue-name image-processor-queue --region ap-south-1 --query QueueUrl --output text)"
echo "Your bucket: image-processor-your-name-123"

cd ..
### Start the Application
```bash
docker-compose up --build
```
### Test It Works
```bash
# Test API
curl http://localhost:8000/

# Upload an image
curl -o test.jpg "https://picsum.photos/800/600"
curl -X POST "http://localhost:8000/upload-image/" -F "file=@test.jpg"

# Check processing status (replace task_id with actual ID from response to know the status)
curl "http://localhost:8000/status/YOUR_TASK_ID"
```
**##Deployment**

aws configure ( enter access key,secret key ,region)

**Create container repositories:**
aws ecr create-repository --repository-name image-processor-api --region ap-south-1
aws ecr create-repository --repository-name image-processor-worker --region ap-south-1
#  Login to ECR
 ```aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 534211282949.dkr.ecr.ap-south-1.amazonaws.com```

#  Build for AMD64 platform 
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

# Update ECS services (triggers deployment)
```aws ecs update-service --cluster image-processor-cluster --service api-service --force-new-deployment
aws ecs update-service --cluster image-processor-cluster --service worker-service --force-new-deployment```

**# 5. Monitor deployment**

```aws ecs describe-services --cluster image-processor-cluster --services api-service worker-service --query 'services[*].[serviceName,runningCount,desiredCount]' --output table```


Live Demo: http://image-processor-alb-1146437516.ap-south-1.elb.amazonaws.com
# Upload image
curl -X POST "http://image-processor-alb-1146437516.ap-south-1.elb.amazonaws.com/upload" -F "file=@test-image.jpg"

