# Deploy a Lambda function-like workload inside an EKS cluster using the Serverless Framework

This guide will walk you through the steps to deploy a Lambda function-like workload inside an EKS cluster using the Serverless Framework. 

## Prerequisites

Before you begin, you will need to have the following installed:

- [Docker](https://www.docker.com/products/docker-desktop)
- [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [AWS CLI](https://aws.amazon.com/cli/)
- [Serverless Framework](https://www.serverless.com/learn/getting-started/)

You will also need:

- An AWS account with permissions to create an EKS cluster and deploy resources to it.
- A Docker Hub account (or an account with another Docker image registry).

## Step 1: Install AWS EKS CLI and serverless Tools
```yaml
$ curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
$ chmod +x ./aws-iam-authenticator
$ sudo mv aws-iam-authenticator /usr/local/bin
$ npm install -g serverless
$ npm install --save serverless-python-requirements serverless-iam-roles-per-function serverless-ecr
```
## Step 2: Configure AWS CLI
```bash
$ aws configure
```
## Step 3: Create a new Serverless service
```bash
$ serverless create --template aws-python3 --name my-service
$ cd my-service
```

## Step 4: Install Serverless plugins
```bash
$ serverless plugin install -n serverless-python-requirements
$ serverless plugin install -n serverless-eks
```
## Step 5: Write Lambda-like function code
Create a new Python file called `handler.py` with the following code:

```python
def handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Hello, World!'
    }
}

## Step 6: Create Dockerfile

Create a file called `Dockerfile` in the root directory of your project and add the following code:

```Dockerfile
FROM python:3.9-slim-buster

# Install the AWS CLI
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN curl -sSL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip /aws

# Set the working directory to /app
WORKDIR /app

# Copy the requirements.txt file to the working directory
COPY requirements.txt .

# Install the dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the code to the working directory
COPY . .

# Set the LAMBDA_TASK_ROOT environment variable
ENV LAMBDA_TASK_ROOT=/var/task

# Define the command to run when the container starts
CMD [ "python", "./handler.py" ]
```

## Step 7:  Build and tag the Docker image

In the root directory of your project, run the following command to build and tag the Docker image:

```bash
docker build -t my-function .
```
## Step 8: Push the Docker image to Amazon ECR
Before pushing the Docker image to Amazon ECR, you need to create an ECR repository. 

After you have created the repository, run the following commands to push the Docker image to ECR:

```bash
# Get the login command from ECR and execute it
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com

# Tag the Docker image with the ECR repository URI
docker tag my-function:latest <account_id>.dkr.ecr.<region>.amazonaws.com/my-function:latest

# Push the Docker image to ECR
docker push <account_id>.dkr.ecr.<region>.amazonaws.com/my-function:latest

```

## Step 9: Deploy the application to EKS

Create a file called serverless.yml in the root directory of your project and add the following code:

```yaml
provider:
  name: aws
  runtime: python3.8
  region: eu-west-1
  stage: dev
  iamRoleStatements:
    - Effect: Allow
      Action:
        - ecr:GetAuthorizationToken
        - ecr:BatchCheckLayerAvailability
        - ecr:GetDownloadUrlForLayer
        - ecr:BatchGetImage
      Resource: '*'
  ecr:
    images:
      my-image:
        path: <account_id>.dkr.ecr.<region>.amazonaws.com/my-function:latest
        configFile: deployment.yaml
       
functions:
  my-function:
    handler: handler.lambda_handler
    events:
      - http:
          path: /
          method: get
    image:
      name: <ECR-repo-uri>:latest

plugins:
  - serverless-python-requirements
  - serverless-iam-roles-per-function
  - serverless-ecr
  - serverless-kubeless
custom:
  kubeless:
    clusterId: <cluster_id>
    namespace: <namespace>
    functionName: my-function
    functionMemorySize: 128Mi

```

## Step 10: Deploy the application

To deploy the application, run the following command in the root directory of your project:

```bash
serverless deploy


## Step 11: Create a Kubernetes deployment manifest
Create a new file named deployment.yml in the root of your service directory with the following content:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  labels:
    app: my-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        - name: my-service
          image: <your-aws-account-id>.dkr.ecr.us-east-1.amazonaws.com/my-function:latest
          ports:
            - containerPort: 8080
    

```

## Step 12: Create a Kubernetes service manifest

Create a new file named service.yml in the root of your service directory with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: LoadBalancer

```

This manifest describes a Kubernetes service that exposes port 80 on the external IP address of the load balancer.

## Step 12: Deploy the Kubernetes manifests
You can deploy the Kubernetes manifests using the following command:
```yaml
kubectl apply -f deployment.yml
kubectl apply -f service.yml
```
This will create a Kubernetes deployment and service for your application.

## Step 13: Trigger the pods on EKS
Now that your application is deployed on EKS, you can trigger the pods by sending an HTTP request to the external IP address of the load balancer. You can use the following command to get the external IP address:

```bash
kubectl get service my-service
```
Once you have the external IP address, you can send an HTTP request to it using a tool like curl:

```bash
curl http://<external-ip>/path/to/your/function
```
Replace <external-ip> with the external IP address of the load balancer, and /path/to/your/function with the path to your function in your code.

And that's it! You have successfully deployed a Python lambda-like workload inside an EKS cluster using the Serverless Framework (sls).
