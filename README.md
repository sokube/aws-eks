# AWS Elastic Kubernetes Service (EKS)

![EKS](https://user-images.githubusercontent.com/58267422/90319616-fd0f7500-df39-11ea-908c-01801a32a654.png)

<br>

# General 

## Amazon Web Services (AWS)

[Amazon Web Services (AWS)](https://aws.amazon.com/what-is-aws/?nc1=h_ls) is the world’s most comprehensive and broadly adopted cloud platform, offering over 175 fully featured services from data centers globally. Millions of customers—including the fastest-growing startups, largest enterprises, and leading government agencies—are using AWS to lower costs, become more agile, and innovate faster.

## AWS Elastic Kubernetes Service

[AWS Elastic Kubernetes Services (EKS)](https://aws.amazon.com/eks/?nc1=h_ls) is a fully managed [Kubernetes](https://kubernetes.io/) service provided by AWS. 

<br>

# Description of project

This tutorial will demonstrate how to create a **AWS Elastic Kubernetes Service (EKS)** managed cluster. This will be done in 3 mains steps :
- __Networking Setup__, in this section we will set up AWS network environment by creating different resources like a Virtual Private Cloud (VPC), Internet Gateway (IGW), Route Table, Routes, Subnets and Security Group (SG).

- __AWS EKS Cluster__, then in this section, we will create IAM Cluster (Control Plane) Role, provision EKS Control Plane, IAM Node Group Role and provision Node Group (Node Group is the name given by AWS to describe group of worker nodes).

- __Cluster Testing__, finally we will deploy a simple web application in the EKS Cluster to verify that it's working as expected.

<br>

# Tutorial Deployment 

To deploy the EKS Cluster we purpose two methods : 

-  [Infrastructure as Code (IaC)](https://github.com/sokube/aws-eks/tree/master/infrastructure-as-code) tutorial with AWS CloudFormation,
-  [AWS Command Line Interface](https://github.com/sokube/aws-eks/tree/master/aws-cli) tutorial with shell tool.

<br>

# Infrastructure 

## EKS Architecture for Control Plane and Worker Node Communication

![EKS communication infra](https://user-images.githubusercontent.com/58267422/101462536-9d9c0180-393c-11eb-9463-ae822e169035.png)

<br>

## EKS Detailed Network Infrastructure

![EKS network schema](https://user-images.githubusercontent.com/58267422/101462571-a5f43c80-393c-11eb-8509-33096c08d462.png)

<br>

## EKS Project Architecture

![EKS infra schema](https://user-images.githubusercontent.com/58267422/101462557-a2f94c00-393c-11eb-885b-cb4ad1367423.png)
