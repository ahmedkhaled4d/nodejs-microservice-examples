---
title: Step-By-Step Instructions To Deploy A Node.Js App To Kubernetes on EKS
author: ahmed khaled
pubDatetime: 2024-09-23T03:42:51Z
slug: deploy-nodejs-app-on-kubernetes-aws-eks
featured: false
draft: false
tags:
  - nodejs
  - aws
  - eks
  - kubernetes
  - deploy
description: "From EC2 to Kubernetes on EKS Step-By-Step Instructions To Deploy A Node.Js App To Kubernetes on EKS"
---

You have this cool app you wrote in Node.js. You're a great developer, and learned how to transform your app into a scalable app with ECS. However, the powers that be have decided that you need to use [Kubernetes](https://kubernetes.io/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks). and you understand the basic building blocks and have drawn the parallels with ECS, but you're not sure how to go from code to app deployed in EKS.

We're going to use the following AWS services:

- **EKS:** A managed Kubernetes cluster, which essentially does the same as ECS but using Kubernetes.
- **Elastic Container Registry (ECR):** A managed container registry for storing, managing, and deploying Docker images.
- **Fargate:** A serverless compute engine for containers that eliminates the need to manage the underlying EC2 instances.

![](https://media.beehiiv.com/cdn-cgi/image/fit=scale-down,format=auto,onerror=redirect,quality=80/uploads/asset/file/da80a7e9-fc40-46fe-9b34-c8df8a9b26c9/EKS_one_pod__1_.png)

### **Solution step by step**

**Note:** The first steps of installing Docker and dockerizing the app are the same as the previous issue. I'm adding them in case you didn't read it, but if you followed them last week, feel free to start from the 6th step, right after pushing the Docker image to ECR.

- **Install Docker on your local machine.**

  Follow the instructions from the official Docker website:

  - Windows: [https://docs.docker.com/desktop/windows/install/](https://docs.docker.com/desktop/windows/install/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)
  - macOS: [https://docs.docker.com/desktop/mac/install/](https://docs.docker.com/desktop/mac/install/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)
  - Linux: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)

- **Create a Dockerfile.**

  In your app's root directory, create a file named "Dockerfile" (no file extension). Use the following as a starting point, adjust as needed.

```
# Use the official Node.js image as the base image
FROM node:latest

# Set the working directory for the app
WORKDIR /app

# Copy package.json and package-lock.json into the container
COPY package*.json ./

# Install the app's dependencies
RUN npm ci

# Copy the app's source code into the container
COPY . .

# Expose the port your app listens on
EXPOSE 3000

# Start the app
CMD ["npm", "start"]
```

- **Build the Docker image and test it locally.**

  While in your app's root directory, build the Docker image. Once the build is complete, start a local container using the new image. Test the app in your browser or with curl or Postman to ensure it's working correctly. If it's not, go back and fix the Dockerfile.

```
docker build -t cool-nodejs-app .
docker run -p 3000:3000 cool-nodejs-app
```

- **Create the ECR registry.**

  First, create a new ECR repository using the AWS Management Console or the AWS CLI.

```
aws ecr create-repository --repository-name cool-nodejs-app
```

- **Push the Docker image to ECR.**

  The command that creates the ECR registry will output the instructions in the output to authenticate Docker with your ECR repository. Follow them. Then, tag and push the image to ECR, replacing {AWSAccountId} and {AWSRegion} with the appropriate values:

```
docker tag cool-nodejs-app:latest {AWSAccountId}.dkr.ecr.{AWSRegion}.amazonaws.com/cool-nodejs-app:latest
docker push {AWSAccountId}.dkr.ecr.{AWSRegion}.amazonaws.com/cool-nodejs-app:latest
```

- **Install and configure AWS CLI, eksctl, and kubectl.**

  Follow the instructions from the official sites:

  - Install the AWS CLI: [https://aws.amazon.com/cli/](https://aws.amazon.com/cli/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)
  - Install kubectl: [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)
  - Install eksctl: [https://github.com/eksctl-io/eksctl](https://github.com/eksctl-io/eksctl?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks)

  After installing it, make sure to configure your AWS CLI with your AWS credentials.

- **Create an EKS Cluster.**  
  You'll need an existing VPC for this (you can use the default one). Create a CloudFormation template named "eks-cluster.yaml" with the following contents. Replace {VPCID} with the ID of your VPC, and {SubnetIDs} with one or more subnet IDs. Then, in the AWS Console, create a new CloudFormation stack using the "eks-cluster.yaml" file as the template.

```
Resources:
  EKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: cool-nodejs-app-eks-cluster
      RoleArn: !GetAtt EKSClusterRole.Arn
      ResourcesVpcConfig:
        SubnetIds: [{SubnetIDs}]
        EndpointPrivateAccess: true
        EndpointPublicAccess: true
      Version: '1.22'

  EKSClusterRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: eks.amazonaws.com
            Action: sts:AssumeRole
      Path: "/"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
        - arn:aws:iam::aws:policy/AmazonEKSServicePolicy

  EKSFargateProfile:
    Type: AWS::EKS::FargateProfile
    Properties:
      ClusterName: !Ref EKSCluster
      FargateProfileName: cool-nodejs-app-fargate-profile
      PodExecutionRoleArn: !GetAtt FargatePodExecutionRole.Arn
      Subnets: [{SubnetIDs}]
      Selectors:
        - Namespace: {Namespace}

  FargatePodExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: 'eks-fargate-pods.amazonaws.com'
            Action: 'sts:AssumeRole'
      Path: "/"
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKS_Fargate_PodExecutionRole_Policy

```

- **Deploy the app to EKS.**

  Create a Kubernetes manifest file named "eks-deployment.yaml" with the following contents. Replace {AWSAccountId} and {AWSRegion} with the appropriate values. Then apply it with `kubectl apply -f eks-deployment.yaml`.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cool-nodejs-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cool-nodejs-app
  template:
    metadata:
      labels:
        app: cool-nodejs-app
    spec:
      containers:
        - name: cool-nodejs-app
          image: {AWSAccountId}.dkr.ecr.{AWSRegion}.amazonaws.com/cool-nodejs-app:latest
          ports:
            - containerPort: 3000
          resources:
            limits:
              cpu: 500m
              memory: 512Mi
            requests:
              cpu: 250m
              memory: 256Mi
      serviceAccountName: fargate-pod-execution-role

---

apiVersion: v1
kind: Service
metadata:
  name: cool-nodejs-app
spec:
  selector:
    app: cool-nodejs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: LoadBalancer
```

- **Set up auto-scaling for the Kubernetes Deployment.**  
  Add the following contents to your "eks-deployment.yaml" file to set up auto-scaling policies based on CPU utilization. Then apply the changes with `kubectl apply -f eks-deployment.yaml`.

```
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: cool-nodejs-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cool-nodejs-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

- **Test the new app.**

  To test the app, find the LoadBalancer's external IP or hostname by looking at the "EXTERNAL-IP" field in the output of the `kubectl get services` command. Paste the IP or hostname in your browser or use curl or Postman. If everything is set up correctly, you should see the same output as when running the app locally. Congrats!

### **Solution explanation**

- **Install Docker on your local machine.**

  Same as last week, Kubernetes runs Docker containers. Docker also lets us run our app locally without depending on the OS, which is irrelevant for a simple test, but really important when your team's computers aren't standardized.

- **Create a Dockerfile.**

  We're using the same Dockerfile as last week. In case you didn't read it, a Dockerfile is a script that tells Docker how to build a Docker image. We specified the base image, copied our app's files, installed dependencies, exposed the app's port, and defined the command to start the app. Writing this when starting development is pretty easy, but doing it for an existing app is harder.

- **Build the Docker image and test it locally.**

  I'm going to repeat one of my favorite phrases in software development: The degree to which you know how your software behaves is the degree to which you've tested it.

- **Create the ECR registry.**

  ECR is a managed container registry that stores Docker container images. It integrates seamlessly with EKS, which is why we're using it here.

- **Push the Docker image to ECR**  
  We build our Docker image, and we push it to the registry so we can pull it from there.
- **Install and configure AWS CLI, eksctl, and kubectl.**

  More tools that we need. The AWS CLI you already know. kubectl is a CLI tool used to manage kubernetes, and it's usually the main way in which you interact with kubernetes. eksctl is a great CLI tool that follows the spirit of kubectl, but which is designed to deal with the parts of EKS that are specific to EKS and not general to all Kubernetes installations. AWS built their own managed Kubernetes, which is EKS, but they (obviously) couldn't modify kubectl, so they built a tool to complement it.

- **Create an EKS Cluster.**  
  A cluster is a collection of worker nodes running our Kubernetes apps. The cluster is managed by the control plane, which includes components like the API server, controller manager, and etcd. Same concept as the ECS cluster, except that with EKS being a managed Kubernetes we can peek under the hood.  
  In our template we're also creating a FargateProfile, which EKS needs so it can run our apps on Fargate (serverless worker nodes, which we use in this issue so we can focus on EKS and not on managing EC2 instances).
- **Deploy the app to EKS.**

  We're creating a YAML file, but this one's not a CloudFormation template. Kubernetes uses “manifest files” to define its resources as code, and our eks-deployment.yaml file is one of those. The key elements are:

  - **Deployment:** Basically, the thing that tells k8s to run our app. More technically, an object that manages a replicated application. It ensures that a specified number of replicas of your application are running at any given time. Deployments are responsible for creating, updating, and rolling back pods. Similar to a Task Definition from ECS.
  - **Service:** Basically, the load balancer of our app. More technically, a logical abstraction on top of one or more pods. It defines a policy by which pods can be accessed, either internally within the cluster or externally. Services can be accessed by a ClusterIP, NodePort, LoadBalancer, or ExternalName. Same concept as a Service in ECS, just some different implementation details. Note that in this case it's going to be a load balancer, but it's not the only option.

  You've probably noticed that we're not creating pods. A pod is one instance of our app executing, so that's what we're really after. However, just like with ECS where we don't create Tasks directly, we don't create Pods directly in Kubernetes.

- **Set up auto-scaling for the Kubernetes Deployment.**  
  We're changing the deployment so it includes the necessary logic to auto-scale. That is, to create and destroy Pods as needed to match the specified metric (in this case CPU usage).
- **Test the new app.**

  You know it's going to work. You test it anyways.

## Best Practices

### Operational Excellence

- **Implement health checks:** Set up health checks for your application in the Kubernetes Service manifest to ensure the load balancer only directs traffic to healthy instances.
- **Use Kubernetes readiness and liveness probes:** Configure readiness and liveness probes for your application container to help detect issues and restart unhealthy containers automatically.
- **Monitor application metrics:** Use Amazon CloudWatch Container Insights to get more metrics from your app.
- **Set up a Service Mesh:** A deployment or two, each running 5 pods, is easy to manage. 10 deployments of (micro)services that need to talk to one another is 100x more complex. A Service Mesh solves the networking part of making pods talk to each other, making it only 50x more complex than a single deployment (it's a big win!)

### Security

- **Use IAM roles:** Set up IAM roles for your Fargate profile or for the EC2 instances, so your resources only have the permissions they need.
- **Restrict network access inside the cluster:** Set up Network Policies to define at the network level what pods can talk to what pods. Basically, firewalls inside the cluster itself.
- **Enable private network access:** Use private subnets for your EKS cluster and configure VPC endpoints for ECR, to ensure that network traffic between your cluster and the ECR registry stays within the AWS network.
- **Use Secrets:** Kubernetes can store secrets in the cluster. Same as what we've been discussing about Secrets Manager, but in this case you don't need another AWS service. You can sync them if you want.
- **Use RBAC:** RBAC = Role-Based Access Controls. It's the same thing we do with AWS IAM Roles, but for Kubernetes resources and actions. Don't give everyone admin, use roles for minimum permissions.

### Reliability

- **Use Horizontal Pod Autoscaler (HPA):** Configure HPA to automatically scale the number of application instances based on CPU utilization, ensuring that the application remains responsive during traffic fluctuations. The scaling that we defined in the Kubernetes manifest only creates more pods, this is what creates more instances for your pods to run on. You use this instead of having your Auto Scaling Group launch instances, because while the ASG has access to instance-level metrics such as actual CPU utilization, HPA has access to Kubernetes metrics such as total requested CPU and memory.
- **Deploy across multiple Availability Zones (AZs):** Ensure that your application's load balancer and Fargate profiles are configured to span multiple AZs for increased availability.
- **Use rolling updates:** Configure your Kubernetes Deployment manifest to perform rolling updates, ensuring that your application remains available during updates and new releases. You do this directly from Kubernetes, [like this](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks).

### Performance Efficiency

- **Optimize resource requests and limits:** Kubernetes defines resource requests and limits. The request is the minimum, and it's guaranteed to the pod (if Kubernetes can't fulfill the request, the pod fails to launch). From there, a pod can ask for more resources up to the limit, and they will be allocated if there are resources available. For example, if you have one node with 2 GB of memory, you deploy pod1 with request: 1 GB and limit: 1.5 GB, pod1 will get 1 GB of memory and can request for more up to 1.5 GB. If you then deploy pod2 with request: 1 GB, Kubernetes will fit both, but now pod1 can't get more memory than it's base 1 GB, because there's no free memory available. Both pod1 and pod2 succeed in launching, because their request can be fulfilled.
- **Monitor performance of the app and cluster:** Use tools like X-Ray and CloudWatch to monitor the performance of your application. Use tools like Prometheus and Grafana to monitor the performance of the cluster (free resources, etc).

### Cost Optimization

- **Right-size the pods:** As always, avoid over-provisioning resources.
- **Consider Savings Plans:** Remember that EKS is priced at $72/month for the cluster alone, plus the EC2 or Fargate pricing for the capacity (the worker nodes). Savings Plans apply to that capacity, just like if EKS wasn't there.
- **Use an Ingress Controller:** Each service of type Load Balancer is a new ALB. Instead of doing that, use a single Ingress Controller per app (you can use one or multiple apps per cluster). The Ingress Controller routes all traffic from outside the cluster, with a single Load Balancer and one or more routes per service.

## Resources

Bad news: the cluster you created with eksctl or with my CloudFormation snippet is **full of security holes**. You should instead use [this free tool](https://github.com/lightspin-tech/eks-creation-engine?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks) to create an opinionated and **very secure cluster**. It's so great I was on the verge of just using it on the solution step by step.

The [official Kubernetes docs](https://kubernetes.io/docs/home/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks) are great as a technical reference, but awful for learning. Instead, check out [this workshop](https://www.eksworkshop.com/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks).

Instead of Horizontal Pod Autoscaler, check out [Karpenter](https://karpenter.sh/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks), which does a better job of managing resources.

To implement Network Policies, check out [Calico](https://www.tigera.io/project-calico/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks). Here's [how to set it up](https://docs.aws.amazon.com/eks/latest/userguide/calico.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks).

Something new: [this platform](https://kubevela.io/?utm_source=simpleaws&utm_medium=referral&utm_campaign=from-ec2-to-kubernetes-on-eks) promises to make shipping applications more enjoyable. I haven't tried it yet.
