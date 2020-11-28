# Prerequisites

## Amazon Web Services

This tutorial leverages the [Amazon Web Services](https://aws.amazon.com/) to provisioning the AWS network infrastructure and AWS EKS Cluster.

## AWS Command Line Interface (CLI)

The [AWS Command Line Interface (CLI)](https://aws.amazon.com/cli/) is a unified tool to manage your AWS services. With just one tool to download and configure, you can control multiple AWS services from the command line and automate them through scripts.
To perform operations and interact with AWS you need to configure basic settings for AWS CLI like security credentials, AWS Region or default output format. You can quickly set up AWS CLI with the [official documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

## AWS CloudFormation

[AWS CloudFormation](https://aws.amazon.com/cloudformation/?nc1=h_ls) provides a common language for you to model and provision AWS and third party application resources in your cloud environment. CloudFormation allows you to use programming languages or a simple text file to model and provision, in an automated and secure manner, all the resources needed for your applications across all regions and accounts. This gives you a single source of truth for your AWS and non-AWS resources.

## AWS Roles 
This project uses [IAM roles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) to assign rights to resources and perform necessary operations.

### IAM Role 

To allow EKS Cluster to perform all necessary operations, we will create and use two roles with specific policies :
- **IAM Cluster Role** : The role that Amazon EKS will use to create AWS resources for Kubernetes clusters.
- **IAM Node Instance Role** : The role attached to a Node Group (worker nodes) group to perform necessary operations.

## AWS Virtual Private Cloud (VPC) 

[Amazon Virtual Private Cloud (Amazon VPC)](https://aws.amazon.com/vpc/) lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define.

## Kubernetes

Knowlege of Kubernetes is required to understand this tutorial. You can learn **Kubernetes** with the [offical documentation](https://kubernetes.io/docs/home/).

<br>

# Tutorial Deployment 

## AWS Region 

For this tutorial we will work in ```us-east-1``` AWS Region. All resources will be created in **US EAST (N.Virginia)** region.

Define AWS Region : 
```shell
export EKS_AWS_REGION="us-east-1"
```

## Networking Setup

### VPC and Route Table Creation

To deploy the EKS Cluster, AWS recommends to create your own dedicated and isolated Virtual Private Cloud (VPC). Create a VPC called "eks-VPC" :

```shell
aws ec2 create-vpc \
  --region $EKS_AWS_REGION \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=eks-VPC},{Key=Project,Value=aws-eks}]'
```

> N.B : The creation of VPC, create also this associated Route Table.

After having created the *eks-VPC*, get the ID of them. We will need several times during this tutorial :

```shell
EKS_VPC_ID=$(aws ec2 describe-vpcs \
  --region $EKS_AWS_REGION \
  --filters "Name=tag:Name,Values=eks-VPC" "Name=tag:Project,Values=aws-eks" \
  --query "Vpcs[*].{VpcId:VpcId}" \
  --output text) \
&& echo $EKS_VPC_ID
```

We will also need the ID of the Route Table to create several resources during this tutorial. Get Route Table ID :

```shell
EKS_ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --region $EKS_AWS_REGION \
  --filters "Name=vpc-id,Values=$EKS_VPC_ID" \
  --query "RouteTables[*].{RouteTableId:RouteTableId}" \
  --output text) \
&& echo $EKS_ROUTE_TABLE_ID
```

To ease searching and filtering is important to tag them. Add tags *Name=eks-RouteTable* and "Project=aws-eks" :

```shell
aws ec2 create-tags \
  --region $EKS_AWS_REGION \
  --resources $EKS_ROUTE_TABLE_ID \
  --tags "Key=Name,Value=eks-RouteTable" "Key=Project,Value=aws-eks"
```

We can view the status of the *eks-RouteTable* is active :

```shell
aws ec2 describe-route-tables \
  --region $EKS_AWS_REGION \
  --route-table-id $EKS_ROUTE_TABLE_ID \
  --query "RouteTables[*].Routes[*].{State:State}" \
  --output text
```

We should see :

```shell
active
```

### Internet gateway

In Amazon Cloud, an Internet Gateway (IGW) is a resource that allows communication between your VPC and internet. This will allow internet access for worker nodes. For more information, you can read the [official documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html). 

Create a Internet gateway named *eks-InternetGateway*"* :

```shell
aws ec2 create-internet-gateway \
  --region $EKS_AWS_REGION \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=eks-InternetGateway},{Key=Project,Value=aws-eks}]'
```

Once the *eks-InternetGateway* is created, we retrieve it's ID for a future usage : 

```shell
EKS_IGW_ID=$(aws ec2 describe-internet-gateways \
  --region $EKS_AWS_REGION \
  --filters "Name=tag:Name,Values=eks-InternetGateway" "Name=tag:Project,Values=aws-eks" \
  --query "InternetGateways[*].{InternetGatewayId:InternetGatewayId}" \
  --output text) \
&& echo $EKS_IGW_ID
```

To associate the previously *eks-InternetGateway* to the *eks-VPC*, we need to attach both resources. Attach the Internet Gateway to the the VPC :

```shell
aws ec2 attach-internet-gateway \
  --region $EKS_AWS_REGION \
  --vpc-id $EKS_VPC_ID \
  --internet-gateway-id $EKS_IGW_ID 
```

We need to create a route toward *eks-InternetGateway* enable external communication (i. e. Internet). Allow internet access :

```shell
aws ec2 create-route \
  --region $EKS_AWS_REGION \
  --route-table-id $EKS_ROUTE_TABLE_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $EKS_IGW_ID 
```

### Subnets creation 

AWS provides a principle of Availability Zone (AZ) to increase High-Availability, Fault-Tolerance and Reliability. For more details you can check the [official documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html). To deploy a EKS Cluster it's mandatory to create at least two subnets in two different AZ. Each worker node will be deployed in different AZ.

We will work in the AWS Region US East (N. Virginia). This AWS region corresponds to the code name "us-east-1". This AWS Region is made up of 6 Availability Zones: 

- us-east-1a 
- us-east-1b
- us-east-1c
- us-east-1d
- us-east-1e
- us-east-1f

For this tutorial, we will use ```us-east-1a``` and ```us-east-1b``` Availability Zones. Define location of subnets :

```shell
export EKS_AVAILABILITY_ZONE_01="us-east-1a"
export EKS_AVAILABILITY_ZONE_02="us-east-1b"
```

Both subnets will be created in the *eks-VPC*. 

Create a first subnet named *eks-PublicSubnet01*, in the ```us-east-1a``` Availability Zone with the *10.0.0.0/24* CIDR block :

```shell
aws ec2 create-subnet \
  --region $EKS_AWS_REGION \
  --vpc-id $EKS_VPC_ID \
  --cidr-block 10.0.0.0/24 \
  --availability-zone $EKS_AVAILABILITY_ZONE_01 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-PublicSubnet01},{Key=Project,Value=aws-eks}]'
```

Create a second subnet named *eks-PublicSubnet02*, in the ```us-east-1b``` Availability Zone with the *10.0.1.0/24* CIDR block :

```shell
aws ec2 create-subnet \
  --region $EKS_AWS_REGION \
  --vpc-id $EKS_VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone $EKS_AVAILABILITY_ZONE_02 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-PublicSubnet02},{Key=Project,Value=aws-eks}]'
```

After the subnets are created, we retrieve their ID and attributes for a future usage.

Get ID of the *eks-PublicSubnet01* and allocate this to the variable :

```shell
EKS_SUBNET_ID_01=$(aws ec2 describe-subnets \
  --region $EKS_AWS_REGION \
  --filters  "Name=tag:Name,Values=eks-PublicSubnet01" "Name=tag:Project,Values=aws-eks" \
  --query "Subnets[*].{ID:SubnetId}" \
  --output text) \
&& echo $EKS_SUBNET_ID_01
```

Get ID of the *eks-PublicSubnet02*"* and allocate this to the variable :

```shell
EKS_SUBNET_ID_02=$(aws ec2 describe-subnets \
  --region $EKS_AWS_REGION \
  --filters  "Name=tag:Name,Values=eks-PublicSubnet02" "Name=tag:Project,Values=aws-eks" \
  --query "Subnets[*].{ID:SubnetId}" \
  --output text) \
&& echo $EKS_SUBNET_ID_02
```

Then we need to associate both subnets to the *eks-RouteTable* :

```shell
aws ec2 associate-route-table \
  --region $EKS_AWS_REGION \
  --subnet-id $EKS_SUBNET_ID_01 \
  --route-table-id $EKS_ROUTE_TABLE_ID

aws ec2 associate-route-table \
  --region $EKS_AWS_REGION \
  --subnet-id $EKS_SUBNET_ID_02 \
  --route-table-id $EKS_ROUTE_TABLE_ID
```

### IP address allocation

By default allocation IP address for each subnet is not enabled and every EC2 instance (worker nodes) created in these subnets wont able to be allocate public IP address. This is an importante feature to internet access for Node Group. 

Activate public IP address allocation for both subnets :

```shell
aws ec2 modify-subnet-attribute \
  --region $EKS_AWS_REGION \
  --subnet-id $EKS_SUBNET_ID_01 \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --region $EKS_AWS_REGION \
  --subnet-id $EKS_SUBNET_ID_02 \
  --map-public-ip-on-launch
```

### Security Groups

[Security Groups (SG)](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) is a set of rules with fine granularity to allow, restrict or deny communication toward a resource. 

Finally, create a Security Group to allow communication between EKS Control Plane and worker nodes : 

```shell
export EKS_SECURITY_GROUP="eks-SecurityGroup"

aws ec2 create-security-group \
  --region $EKS_AWS_REGION \
  --group-name $EKS_SECURITY_GROUP \
  --description "Cluster communication with worker nodes" \
  --vpc-id $EKS_VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=eks-SecurityGroup},{Key=Project,Value=aws-eks}]' 
```

After creation, retrieve the ID and attributes for a future usage :

```shell
EKS_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --region $EKS_AWS_REGION \
  --filters "Name=tag:Name,Values=eks-SecurityGroup" "Name=tag:Project,Values=aws-eks" \
  --query 'SecurityGroups[*].{GroupId:GroupId}' \
  --output text) \
&& echo $EKS_SECURITY_GROUP_ID
```

**The network implementation is done. This is an important section because it allow to create your own isolated and secure network for the EKS Cluster. It show also how to allow communication between the VPC and several subnets.**


You can view the network architecture after all network resources creation : 


![AWS network schema](https://user-images.githubusercontent.com/58267422/99916762-4d059100-2d0c-11eb-9439-0a9dea74cdad.png)

<br> 


## AWS EKS Cluster

Retrieve source code on [SoKube](https://www.sokube.ch/) GitHub repository : 

```shell
git clone https://github.com/sokube/aws-eks.git $HOME/aws-eks
cd $HOME/aws-eks/aws-cli
```

### SSH Key pair

In AWS, authentication with a key pair is a privileged method. A key pair is a set of security credentials that you use to prove your identity when connecting to an instance. AWS stores the public key and you store the private key in a secure place. Firstly, we need to create SSH key pair to allow you connect to your worker nodes instances. 

Create "my-eks-key" key pair :

```shell
export EKS_KEY_PAIR_NAME="my-eks-key"

aws ec2 create-key-pair \
  --region $EKS_AWS_REGION \
  --key-name $EKS_KEY_PAIR_NAME \
  --tag-specifications 'ResourceType=key-pair,Tags=[{Key=Name,Value=eks-key-pair},{Key=Project,Value=aws-eks}]' \
  --output text \
  --query 'KeyMaterial' >  $HOME/aws-eks/aws-cli/eks.id_rsa
```

### EKS IAM Cluster Role

Kubernetes clusters managed by Amazon EKS make calls to other AWS services on your behalf to manage the resources that you use with the service. Before you can create Amazon EKS clusters, you must create/declare an IAM role with the IAM policies __AmazonEKSClusterPolicy__ : 

To create this role we will use AWS CloudFormation. CloudFormation is very efficient provisioning tool, similar to Terraform but dedicated to AWS.

Define the name of the EKS CloudFormation Stack Role, allocate this to a variable and then create a *EKSClusterRole* with CloudFormation :

```shell
export EKS_CLUSTER_ROLE_STACK_NAME="EKSClusterRole"

aws cloudformation create-stack \
  --region $EKS_AWS_REGION \
  --stack-name $EKS_CLUSTER_ROLE_STACK_NAME \
  --template-body file://$HOME/aws-eks/aws-cli/eks-cluster-role.yaml \
  --tags Key=Project,Value=aws-eks \
  --capabilities CAPABILITY_IAM
```

> It usually takes about 2 minutes for the cluster role to get created.

Once created, get the Amazon Resource Name (ARN) of the *EKSClusterRole* and allocate this to a variable for a future usage :

```shell
EKS_CLUSTER_ROLE_ARN=$(aws cloudformation describe-stacks \
  --region $EKS_AWS_REGION \
  --stack-name $EKS_CLUSTER_ROLE_STACK_NAME \
  --query "Stacks[*].Outputs[*].{Output:OutputValue}" \
  --output text) \
&& echo $EKS_CLUSTER_ROLE_ARN
```

The result should look like:

```shell
arn:aws:iam::123456789123:role/EKSClusterRole-eksClusterRole-1N4G0KR302L5Y
```

### EKS Control Plane Provisioning

After, having created IAM Cluster Role *EKSClusterRole*  We will create a EKS Control Plane with the following characteristics :

- AWS Region : us-east-1
- EKS Cluster Name : EKS
- Kubernetes version : 1.18

Define and allocate previously characteristics to a variables :

```shell
export EKS_AWS_REGION="us-east-1"
export EKS_CLUSTER_NAME="EKS"
export EKS_CLUSTER_VERSION="1.18"
```

Create a EKS Control Plane :

```shell
aws eks create-cluster \
   --region $EKS_AWS_REGION \
   --name $EKS_CLUSTER_NAME \
   --kubernetes-version $EKS_CLUSTER_VERSION \
   --role-arn $EKS_CLUSTER_ROLE_ARN \
   --resources-vpc-config subnetIds=$EKS_SUBNET_ID_01,$EKS_SUBNET_ID_02,securityGroupIds=$EKS_SECURITY_GROUP_ID \
   --tags Key=Project,Value=aws-eks
```

You can check the cluster creation status with the following command. Until this loop is complete, the EKS cluster is not ready.

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do
  echo -e "EKS Cluster status : CREATING \n"
  if [ $(aws eks --region $EKS_AWS_REGION describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.status" --output text) == ACTIVE ]
  then
    echo -e "EKS Cluster status : ACTIVE \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```

During the creation of the EKS Control plane we should see :

```shell
EKS Cluster status : CREATING 

EKS Cluster status : CREATING 

EKS Cluster status : CREATING 
...

EKS Cluster status : ACTIVE

started at : 23:25:29
finished at : 23:39:15
Total time : 0 h 13 min 46 sec
```

> Please note that a control plane can take around 10-15 minutes to complete.

After the loop either terminated you can ensure the EKS Control Plane is successfully created and ```ACTIVE``` :

```shell
aws eks \
  --region $EKS_AWS_REGION describe-cluster \
  --name $EKS_CLUSTER_NAME \
  --query "cluster.status"
```

### EKS IAM Node Group Role

The Amazon EKS node kubelet daemon makes calls to AWS APIs on your behalf. Nodes receive permissions for these API calls through an IAM instance profile and associated policies. Before you can launch nodes and register them into a cluster, you must create an IAM role for those nodes to use when they are launched. This requirement applies to nodes launched with the Amazon EKS optimized AMI provided by Amazon, or with any other node AMIs that you intend to use. Before you create nodes, you must create an IAM role with the following IAM policies:

- __AmazonEKSWorkerNodePolicy__
- __AmazonEC2ContainerRegistryReadOnly__

The __AmazonEKS_CNI_Policy__ policy must be attached to either this role or to a different role that is mapped to the aws-node Kubernetes service account. We recommend assigning the policy to the role associated to the Kubernetes service account instead of assigning it to this role. For more information, see Walkthrough: Updating the VPC CNI plugin to use IAM roles for service accounts.

Define the name of the Node Group Stack CloudFormation role  , allocate this to a variable and create a *EKSNodeInstanceRole* with CloudFormation :

```shell
export EKS_NODE_GROUP_ROLE_STACK_NAME="EKSNodeInstanceRole"

aws cloudformation create-stack \
  --region $EKS_AWS_REGION \
  --stack-name $EKS_NODE_GROUP_ROLE_STACK_NAME \
  --tags Key=Project,Value=aws-eks \
  --template-body file://$HOME/aws-eks/aws-cli/eks-nodegroup-role.yaml \
  --capabilities CAPABILITY_IAM
```

> Again, the creation of the Node Group role should take around 2 minute.

Then get the Amazon Resource Name (ARN) of the *EKSNodeInstanceRole* previously created and allocate this to a variable for future usage :

```shell
EKS_NODE_GROUP_ROLE_ARN=$(aws cloudformation describe-stacks \
  --region $EKS_AWS_REGION \
  --stack-name $EKS_NODE_GROUP_ROLE_STACK_NAME \
  --query "Stacks[*].Outputs[*].{Output:OutputValue}" \
  --output text) \
&& echo $EKS_NODE_GROUP_ROLE_ARN
```

The result should look like:

```shell
arn:aws:iam::123456789123:role/EKSNodeInstanceRole-NodeInstanceRole-1JLY6NOH2LN1X
```

### EKS Node Group Provisioning

After successfully creation of EKS control plane we will create the Node Group with the following characteristics :

- Name of the Node Group : NodeGroup01
- Type of AMI of the Node Group instance : AL2_x86_64
- Type of instance of the Node Group : t3.medium
- Amount of disk size : 20 GB

Define and allocate previously characteristics to a variables :

```shell
export EKS_NODE_GROUP_NAME="NodeGroup01"
export EKS_NODE_GROUP_AMI_TYPE="AL2_x86_64"
export EKS_NODE_GROUP_INSTANCE_TYPE="t3.medium"
export EKS_NODE_GROUP_DISK_SIZE="20"
```

Now we can deploy the node group instance :

```shell
aws eks create-nodegroup \
  --region $EKS_AWS_REGION \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_NODE_GROUP_NAME \
  --subnets "$EKS_SUBNET_ID_01" "$EKS_SUBNET_ID_02" \
  --node-role $EKS_NODE_GROUP_ROLE_ARN \
  --ami-type $EKS_NODE_GROUP_AMI_TYPE \
  --instance-types $EKS_NODE_GROUP_INSTANCE_TYPE \
  --disk-size $EKS_NODE_GROUP_DISK_SIZE \
  --remote-access ec2SshKey=$EKS_KEY_PAIR_NAME \
  --tags Key=Project,Value=aws-eks
```

Again, You can check the node groups creation status with the following command. Until this loop is complete, the node group is not ready.

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do
  echo -e "EKS Node Group status : CREATING \n"
  if [ $(aws eks --region $EKS_AWS_REGION describe-nodegroup --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_NODE_GROUP_NAME --query "nodegroup.status" --output text) == ACTIVE ]
  then
    echo -e "EKS Node Group status : ACTIVE \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```

> Please note that a control plane can take around 5 minutes to complete.

During the creation of the EKS Node Group we should see :

```shell
EKS Node Group status : CREATING 

EKS Node Group status : CREATING 

EKS Node Group status : CREATING 

...

EKS Node Group status : ACTIVE 

started at : 23:41:28
finished at : 23:42:43
Total time : 0 h 1 min 15 sec
```


After the loop either terminated you can verify the EKS Node Groups is successfully created and ```ACTIVE``` :

```shell
aws eks \
  --region $EKS_AWS_REGION describe-nodegroup \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_NODE_GROUP_NAME \
  --query "nodegroup.status" \
  --output text
```

:white_check_mark:  **Congratulations your EKS Cluster is functional and ready to deploy workload. The next step is ensure this one is working as expected.**

<br>

## EKS Cluster testing

### Kubernetes configuration file 

To test the EKS Cluster, the first step is to generate and retrieve the kubernetes configuration file with credentials to interact with the EKS Cluster through the Kubernetes API. Generate and retrieve the kubeconfig :

```shell
aws eks \
  --region $EKS_AWS_REGION update-kubeconfig \
  --name $EKS_CLUSTER_NAME
```
We should see :

```shell
Updated context arn:aws:eks:us-east-1:123456789123:cluster/EKS in /home/admin/.kube/config
```

> The kubeconfig file is automatically merged to the "$HOME/.kube/config"

You can now use kubectl command to operate the EKS Cluster. For example, you can retrieve informations about worker nodes :

```shell
kubectl get node
```

We should see :
```shell
NAME                      STATUS ROLES  AGE   VERSION
ip-10-0-0-89.ec2.internal Ready  <none> 5m21s v1.18.9-eks-d1db3c
ip-10-0-1-53.ec2.internal Ready  <none> 5m35s v1.18.9-eks-d1db3c
```

### Application Deployment

You can interact with the EKS cluster. You will deploy a pod that hosting a simple Node.js web application to test that EKS cluster works as expected. Right after, we will expose this pod by Kubernetes LoadBalancer service to allow external users access this application. Indeed Kubernetes provides a managed LoadBalancers services (for cloud platforms) that automatically create Load Balancers as needed. For more information you can read the [official documentation](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer).

Create a *app-shark* pod :

```shell
kubectl run app-shark \
  --image=sokubedocker/shark-application:eks \
  --restart=Never

pod/app-shark created
```

to be agile and used in different environments

Create a LoadBalancer service on port 8080 to expose the *app-shark* pod :

```shell
kubectl expose pod app-shark --type=LoadBalancer --port=8080

service/app-shark exposed
```
> The shark-app container in the pod is using port 8080.


Kubernetes is a fully Cloud Native tool and the previous kubectl command triggers the automated creation of an [Elastic Load Balancer (ELB)](https://aws.amazon.com/elasticloadbalancing/?nc1=h_ls&elb-whats-new.sort-by=item.additionalFields.postDateTime&elb-whats-new.sort-order=desc). The ELB creation takes usually around 5 minutes before it can be used.

Waiting the ELB either available we can retrieve informations about created resources :

```shell
kubectl get pod,svc -o wide 
```

To show and reach the *shark-app* you will need to get the AWS ELB DNS Name. The *app-shark* application will be available on this address. Get the DNS Name of the ELB :

```shell
EKS_ELB_HOSTNAME=$(kubectl get svc app-shark -o jsonpath='{.status.loadBalancer.ingress[*].hostname}') \
&& echo $EKS_ELB_HOSTNAME
```

We should see : 

```shell
a2dd31cb63d604165bf3f464b36d626f-1015502290.us-east-1.elb.amazonaws.com
```

> Wait around 5 minutes to the Elastic Load Balancer is up…

Once the ELB is created. we can either use : 

- curl (result is less pretty) :

```shell
curl http://$EKS_ELB_HOSTNAME:8080
```

- Web browser (better solution) 

Paste the DNS Name *http://a2dd31cb63d604165bf3f464b36d626f-1015502290.us-east-1.elb.amazonaws.com:8080* in the web browser and add port 8080 to reach shark application.

We should see:

![shark-app](https://user-images.githubusercontent.com/58267422/100019516-768eed00-2dde-11eb-9cf7-1d4a84a75792.png)


The *shak-app* is well reachable with the DNS name of the ELB. The EKS Cluster is working.

In the current article, we saw how to create a highly available, fault-tolerant EKS managed cluster with two worker nodes and network isolation. Now you can go a step further and find what is hiding this under the hood with the power of Kubernetes and AWS. For example, you can test how to implement RBAC using AWS IAM...

<br>

# Freeing Resources

## Information

If you regularly use AWS, you know the famous :

> "Pay only what you use"

Keep in mind that resources, even idle, are provisioned and billed. To avoid expensive bills, make sure to regularly cleanup your test / demo clusters.

> For information a EKS cluster with these resources cost around 5 $ per day.

## Resources deletion

### Define EKS Cluster resources  

```shell

export EKS_AWS_REGION="us-east-1" 
export EKS_CLUSTER_NAME="EKS"
export EKS_NODE_GROUP_NAME="NodeGroup01"
export EKS_CLUSTER_ROLE_STACK_NAME="EKSClusterRole"
export EKS_NODE_GROUP_ROLE_STACK_NAME="EKSNodeInstanceRole"
export EKS_KEY_PAIR_NAME="my-eks-key"
EKS_ELB_NAME=$(echo $EKS_ELB_HOSTNAME | awk -F "-" '{ print $1 }')
EKS_VPC_ID=$(aws ec2 describe-vpcs --region $EKS_AWS_REGION --filters "Name=tag:Name,Values=eks-VPC" "Name=tag:Project,Values=aws-eks" --query "Vpcs[*].{VpcId:VpcId}" --output text)
EKS_IGW_ID=$(aws ec2 describe-internet-gateways --region $EKS_AWS_REGION --filters "Name=tag:Name,Values=eks-InternetGateway" "Name=tag:Project,Values=aws-eks" --query "InternetGateways[*].{InternetGatewayId:InternetGatewayId}" --output text)
EKS_ROUTE_TABLE_ID=$(aws ec2 describe-route-tables --region $EKS_AWS_REGION --filters "Name=vpc-id,Values=$EKS_VPC_ID" --query "RouteTables[*].{RouteTableId:RouteTableId}" --output text)
EKS_SUBNET_ID_01=$(aws ec2 describe-subnets --region $EKS_AWS_REGION --filters  "Name=tag:Name,Values=eks-PublicSubnet01" "Name=tag:Project,Values=aws-eks" --query "Subnets[*].{ID:SubnetId}" --output text)
EKS_SUBNET_ID_02=$(aws ec2 describe-subnets --region $EKS_AWS_REGION --filters  "Name=tag:Name,Values=eks-PublicSubnet02" "Name=tag:Project,Values=aws-eks" --query "Subnets[*].{ID:SubnetId}" --output text)
EKS_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups --region $EKS_AWS_REGION --filters "Name=tag:Name,Values=eks-SecurityGroup" "Name=tag:Project,Values=aws-eks" --query 'SecurityGroups[*].{GroupId:GroupId}' --output text)
```
<br>

### EKS Node Groups deletion

```shell
aws eks delete-nodegroup \
  --region $EKS_AWS_REGION \
  --cluster-name $EKS_CLUSTER_NAME \
  --nodegroup-name $EKS_NODE_GROUP_NAME
```

> Note : This operation can take around 5-15 minutes

You can get the EKS Node Group deletion status : 

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do 
  echo -e "EKS Node Group status : DELETING  \n"
  aws eks --region $EKS_AWS_REGION describe-nodegroup --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_NODE_GROUP_NAME --query "nodegroup.status" --output text &>/dev/null
  if [[ "$?" -ne 0 ]]; then
    echo -e "EKS Node Group status : SUCCESSFULLY DELETED \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```
We should see :

```shell
DELETING
EKS Node Group status : DELETING  

DELETING
EKS Node Group status : DELETING  

DELETING
EKS Node Group status : DELETING  

...

EKS Node Group status : SUCCESSFULLY DELETED 

started at : 14:18:55
finished at : 14:31:22
Total time : 0 h 12 min 27 sec
```

### EKS Cluster deletion 

```shell
aws eks delete-cluster --region $EKS_AWS_REGION --name $EKS_CLUSTER_NAME
```
> Note : This operation can take around 5-15 minutes

You can get the EKS Cluster deletion status : 

```shell
started_date=$(date |  awk '{print $4}')
start=`date +%s`
while true; do
  echo -e "EKS Cluster status : DELETING \n"
  aws eks --region $EKS_AWS_REGION describe-cluster --name $EKS_CLUSTER_NAME --query "cluster.status" --output text --output text &>/dev/null
  if [[ "$?" -ne 0 ]]; then
    echo -e "EKS Cluster status : SUCCESSFULLY DELETED \n"
    end=`date +%s`
    runtime=$((end-start))
    finished_date=$(date |  awk '{print $4}')
    echo "started at :" $started_date 
    echo "finished at :" $finished_date
    hours=$((runtime / 3600)); minutes=$(( (runtime % 3600) / 60 )); seconds=$(( (runtime % 3600) % 60 )); echo "Total time : $hours h $minutes min $seconds sec"
    break
  fi
done
```

We should see : 

```shell
EKS Cluster status : DELETING  

EKS Cluster status : DELETING 

EKS Cluster status : DELETING  

...

EKS Cluster status : SUCCESSFULLY DELETED 

started at : 14:18:55
finished at : 14:31:22
Total time : 0 h 12 min 27 sec
```

### EKS Security Groups deletion

```shell
aws ec2 delete-security-group --region $EKS_AWS_REGION --group-id  $EKS_SECURITY_GROUP_ID
```

### EKS Node Group Role deletion 

```shell
aws cloudformation delete-stack --region $EKS_AWS_REGION --stack-name $EKS_NODE_GROUP_ROLE_STACK_NAME
```

### EKS Cluster Role deletion

```shell
aws cloudformation delete-stack --region $EKS_AWS_REGION --stack-name $EKS_CLUSTER_ROLE_STACK_NAME
```

### Elastic Load Balancer deletion

```shell
aws elb delete-load-balancer --region $EKS_AWS_REGION --load-balancer-name $EKS_ELB_NAME
```

### Internet Gateway detachment

```shell
aws ec2 detach-internet-gateway --region $EKS_AWS_REGION --internet-gateway-id $EKS_IGW_ID --vpc-id $EKS_VPC_ID
```

### Internet Gateway deletion

```shell
aws ec2 delete-internet-gateway --region $EKS_AWS_REGION --internet-gateway-id $EKS_IGW_ID 
```

### Route deletion from EKS Route Table 

```shell
aws ec2 delete-route --region $EKS_AWS_REGION --route-table-id $EKS_ROUTE_TABLE_ID --destination-cidr-block 0.0.0.0/0 
```

### Subnets deletion

```shell
aws ec2 delete-subnet --region $EKS_AWS_REGION --subnet-id $EKS_SUBNET_ID_01

aws ec2 delete-subnet --region $EKS_AWS_REGION --subnet-id $EKS_SUBNET_ID_02
```

### Security Groups deletion

```shell
for i in `aws ec2 describe-security-groups --region $EKS_AWS_REGION --filters Name=vpc-id,Values="$EKS_VPC_ID" | grep sg- | sed -E 's/^.*(sg-[a-z0-9]+).*$/\1/' | sort | uniq`; \
do 
  aws ec2 delete-security-group --region $EKS_AWS_REGION --group-id $i; 
done

```
> Note : You can have a error message similar to  "An error occurred (CannotDelete) when calling the DeleteSecurityGroup operation: the specified group: "sg-xxxxxxxxxxxxxxxx" name: "default" cannot be deleted by a user"


> This point is not blocking for the destruction of resources linked to EKS

### VPC deletion

```shell
aws ec2 delete-vpc --region $EKS_AWS_REGION --vpc-id $EKS_VPC_ID 
```

### Key pair deletion

```shell
aws ec2 delete-key-pair --region $EKS_AWS_REGION --key-name $EKS_KEY_PAIR_NAME
```

### Private key file deletion

```shell
rm -rf $HOME/aws-eks/aws-cli/eks.id_rsa
```

### Kubeconfig file deletion

```shell
rm -fr $HOME/.kube/config
```

### aws-eks folder deletion

```shell
rm -fr $HOME/aws-eks
```