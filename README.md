## Lesson 2: Kustomize Structure and Resource Management

This documentation explains how to structure a Kustomize project and create/manage Kubernetes resources using bases and overlays.

##Lesson 2.1: Kustomize Structure and Concepts

**Objective**

Understand the directory structure, files, and key concepts of Kustomize.

**Kustomize Directory Structure**

Kustomize uses a base and overlay pattern:

```kustomize-project/
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
└── overlays/
    └── dev/
        ├── kustomization.yaml
```

**Key Components**

**1. Base Directory**

- Contains common and default Kubernetes resources

- Environment-agnostic

- Reusable across multiple environments

**2. Overlays Directory**

- Contains environment-specific customizations

- Examples: dev, staging, prod

**3. kustomization.yaml**

- Core configuration file for Kustomize

- Declares:

  - Resources

   - Bases

  - Patches

  - Environment-specific changes

 ### Create the Base Structure
**Step 1: Create Directories**

```
mkdir -p AWS-KUSTOMIZE/base
mkdir -p AWS-KUSTOMIZE/overlays/dev
cd AWS-KUSTOMIZE
```
### Lesson 2.2: Creating and Managing Resources
**Objective**

Learn how to define and manage Kubernetes resources using Kustomize.

**Step 1: Define a Basic Deployment (Base)**

Create a file called deployment.yaml inside the base directory.

base/deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

Step 2: Create Base Kustomization File

 base/kustomization.yaml
 
 ```
 resources:
  - deployment.yaml
```
### Step 3: Create Overlay Kustomization (Dev)

overlays/dev/kustomization.yaml

```
resources:
  - ../../base
```

Step 4: Build the Kustomize Configuration

Run the following command from the project root:

```
kustomize build overlays/dev
```

### Apply to Kubernetes Cluster

If you have a Kubernetes cluster configured:

```
 kustomize build overlays/dev | kubectl apply -f -
```

##  Leveraging AWS: Using Amazon EKS with Kustomize 

This section demonstrates how to deploy the Kustomize-based Kubernetes configuration to **Amazon EKS**.


## Prerequisites

Before starting, ensure you have:

- An AWS account
- `kubectl` installed
- `kustomize` (optional, `kubectl` includes it)
- Internet access to AWS services

## Step 1: Set Up Your AWS Account and CLI

### 1. Create an AWS Account

If you don’t already have an AWS account, create one via the **AWS Management Console**.

### 2. Install AWS CLI

Install the AWS CLI using the official installation guide.

Verify installation:

   ```bash
    aws --version
  ```
3. Configure AWS CLI

  Run the following command and provide your credentials:
  
  ```
    aws configure
  ```

You will be prompted for:

- AWS Access Key ID

- AWS Secret Access Key

- Default region (e.g., us-east-1)

- Output format (optional)  

### Step 2: Install and Configure eksctl

eksctl is a CLI tool that simplifies EKS cluster creation.

**1. Install** eksctl

Follow the installation instructions from the official eksctl GitHub repository.

Verify installation:
 
 ```
 eksctl version
```

**2. Configure** eksctl

No additional configuration is required.
It automatically uses your AWS CLI credentials.

### Step 3: Create an EKS Cluster
**1. Create the Cluster**
  
  ### STEP 1: Create the VPC (Network)

EKS must run inside a VPC.
Using a defult eks vpc use the aws eks documentation

Download the link below:

```
 https://s3.us-west-2.amazonaws.com/amazon-eks/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
 ```
Run the following command:
  Create a file called eks-vpc.yaml
  
  ```
   ---
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Amazon EKS Sample VPC - Private and Public subnets'

Parameters:

  VpcBlock:
    Type: String
    Default: 192.168.0.0/16
    Description: The CIDR range for the VPC. This should be a valid private (RFC 1918) CIDR range.

  PublicSubnet01Block:
    Type: String
    Default: 192.168.0.0/18
    Description: CidrBlock for public subnet 01 within the VPC

  PublicSubnet02Block:
    Type: String
    Default: 192.168.64.0/18
    Description: CidrBlock for public subnet 02 within the VPC

  PrivateSubnet01Block:
    Type: String
    Default: 192.168.128.0/18
    Description: CidrBlock for private subnet 01 within the VPC

  PrivateSubnet02Block:
    Type: String
    Default: 192.168.192.0/18
    Description: CidrBlock for private subnet 02 within the VPC

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      -
        Label:
          default: "Worker Network Configuration"
        Parameters:
          - VpcBlock
          - PublicSubnet01Block
          - PublicSubnet02Block
          - PrivateSubnet01Block
          - PrivateSubnet02Block

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock:  !Ref VpcBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
      - Key: Name
        Value: !Sub '${AWS::StackName}-VPC'

  InternetGateway:
    Type: "AWS::EC2::InternetGateway"

  VPCGatewayAttachment:
    Type: "AWS::EC2::VPCGatewayAttachment"
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Public Subnets
      - Key: Network
        Value: Public

  PrivateRouteTable01:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Private Subnet AZ1
      - Key: Network
        Value: Private01

  PrivateRouteTable02:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
      - Key: Name
        Value: Private Subnet AZ2
      - Key: Network
        Value: Private02

  PublicRoute:
    DependsOn: VPCGatewayAttachment
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PrivateRoute01:
    DependsOn:
    - VPCGatewayAttachment
    - NatGateway01
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable01
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway01

  PrivateRoute02:
    DependsOn:
    - VPCGatewayAttachment
    - NatGateway02
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable02
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway02

  NatGateway01:
    DependsOn:
    - NatGatewayEIP1
    - PublicSubnet01
    - VPCGatewayAttachment
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt 'NatGatewayEIP1.AllocationId'
      SubnetId: !Ref PublicSubnet01
      Tags:
      - Key: Name
        Value: !Sub '${AWS::StackName}-NatGatewayAZ1'

  NatGateway02:
    DependsOn:
    - NatGatewayEIP2
    - PublicSubnet02
    - VPCGatewayAttachment
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt 'NatGatewayEIP2.AllocationId'
      SubnetId: !Ref PublicSubnet02
      Tags:
      - Key: Name
        Value: !Sub '${AWS::StackName}-NatGatewayAZ2'

  NatGatewayEIP1:
    DependsOn:
    - VPCGatewayAttachment
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc

  NatGatewayEIP2:
    DependsOn:
    - VPCGatewayAttachment
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc

  PublicSubnet01:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Subnet 01
    Properties:
      MapPublicIpOnLaunch: true
      AvailabilityZone:
        Fn::Select:
        - '0'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PublicSubnet01Block
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: !Sub "${AWS::StackName}-PublicSubnet01"
      - Key: kubernetes.io/role/elb
        Value: 1

  PublicSubnet02:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Subnet 02
    Properties:
      MapPublicIpOnLaunch: true
      AvailabilityZone:
        Fn::Select:
        - '1'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PublicSubnet02Block
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: !Sub "${AWS::StackName}-PublicSubnet02"
      - Key: kubernetes.io/role/elb
        Value: 1

  PrivateSubnet01:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Subnet 03
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '0'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PrivateSubnet01Block
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: !Sub "${AWS::StackName}-PrivateSubnet01"
      - Key: kubernetes.io/role/internal-elb
        Value: 1

  PrivateSubnet02:
    Type: AWS::EC2::Subnet
    Metadata:
      Comment: Private Subnet 02
    Properties:
      AvailabilityZone:
        Fn::Select:
        - '1'
        - Fn::GetAZs:
            Ref: AWS::Region
      CidrBlock:
        Ref: PrivateSubnet02Block
      VpcId:
        Ref: VPC
      Tags:
      - Key: Name
        Value: !Sub "${AWS::StackName}-PrivateSubnet02"
      - Key: kubernetes.io/role/internal-elb
        Value: 1

  PublicSubnet01RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet01
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet02RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet02
      RouteTableId: !Ref PublicRouteTable

  PrivateSubnet01RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet01
      RouteTableId: !Ref PrivateRouteTable01

  PrivateSubnet02RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet02
      RouteTableId: !Ref PrivateRouteTable02

  ControlPlaneSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Cluster communication with worker nodes
      VpcId: !Ref VPC

Outputs:

  SubnetIds:
    Description: Subnets IDs in the VPC
    Value: !Join [ ",", [ !Ref PublicSubnet01, !Ref PublicSubnet02, !Ref PrivateSubnet01, !Ref PrivateSubnet02 ] ]

  SecurityGroups:
    Description: Security group for the cluster control plane communication with worker nodes
    Value: !Join [ ",", [ !Ref ControlPlaneSecurityGroup ] ]

  VpcId:
    Description: The VPC Id
    Value: !Ref VPC
```
Create the stack:
 
 ```
  aws cloudformation create-stack \
  --stack-name eks-vpc \
  --template-body file://eks-vpc.yaml
```

Check stack details
 
 ```
   aws cloudformation describe-stacks \
  --stack-name eks-vpc \
  --region us-east-1
```

 ### STEP 2: Create the EKS Cluster

Create eks-cluster.yaml

```
 AWSTemplateFormatVersion: '2010-09-09'
Description: EKS Cluster

Parameters:
  ClusterName:
    Type: String
    Default: my-eks-cluster

Resources:
  EksRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: eks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

  EksCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: !Ref ClusterName
      RoleArn: !GetAtt EksRole.Arn
      ResourcesVpcConfig:
        SubnetIds:
          - subnet-xxxxxxxx
          - subnet-yyyyyyyy
```
Replace with the subnet that has ben created already

Create the stack:

```
aws cloudformation create-stack \
  --stack-name eks-cluster \
  --template-body file://eks-cluster.yaml \
  --capabilities CAPABILITY_NAMED_IAM
  ```
  ### Check the cluster status
  
  ```
   aws eks describe-cluster \
  --name my-simple-eks \
  --region us-east-1 \
  --query "cluster.status"
```

### STEP 4: Create a Node Group (Workers)

### 1 Create a proper IAM role for nodes

Example using CLI:

```
 aws iam create-role \
    --role-name EKSNodeRole \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "ec2.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }'
```
### 2 Attach required policies to the role

```
  aws iam attach-role-policy --role-name EKSNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name EKSNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
aws iam attach-role-policy --role-name EKSNodeRole --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
```
**3 Get the role ARN**

```
aws iam get-role --role-name EKSNodeRole

```
Copy the Arn field (it will look like arn:aws:iam::707913648704:role/EKSNodeRole)

**4 Create Node Group with correct ARN**

```
aws eks create-nodegroup \
  --cluster-name my-simple-eks \
  --nodegroup-name my-nodegroup \
  --scaling-config minSize=1,maxSize=3,desiredSize=2 \
  --disk-size 20 \
  --subnets subnet-096d6adaf2387253f subnet-0d833d50891663420 \
  --instance-types t3.medium \
  --ami-type AL2_x86_64 \
  --node-role arn:aws:iam::707913648704:role/EKSNodeRole
```

Check node group status:

```
aws eks describe-nodegroup \
  --cluster-name my-simple-eks \
  --nodegroup-name my-nodegroup \
  --region us-east-1
```
Then run:
 
 ```
  kubectl get nodes
 ```
  
  ```
  kubectl apply -k overlays/dev/
```
Notes:

- Cluster creation may take 10–15 minutes

- This creates an EKS cluster with 3 worker nodes

2. Verify Cluster Creation

Once completed, verify access:

```
kubectl get nodes
```

### Step 4: Deploying Kustomize Configurations to EKS
1. Prepare Your Kustomize Project

Ensure your project structure includes:

```
 base/
overlays/
  ├── dev/
  └── prod/
```
### 2. Deploy Dev Environment
```
kubectl apply -k overlays/dev
```

### 3. Deploy Prod Environment

```
kubectl apply -k overlays/prod
```

### 4. Verify Deployment 

```
kubectl get deployments
kubectl get pods
```
---

## Step 5: Verify and Troubleshoot Deployment

After deploying your Kustomize configurations to Amazon EKS, verify that all resources are running as expected.

---

### 1. Check Deployed Resources

Run the following command to view all deployed Kubernetes resources:

```bash
kubectl get all
```
This command lists:

- Pods

- Services

- Deployments

- ReplicaSets

You should see the resources defined in your Kustomize base and overlays.