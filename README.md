# AWS Elastic Kubernetes Service (EKS)

![EKS](https://user-images.githubusercontent.com/58267422/90319616-fd0f7500-df39-11ea-908c-01801a32a654.png)

<br>

# General 

## Amazon Web Services (AWS)

[Amazon Web Services (AWS)](https://aws.amazon.com/what-is-aws/?nc1=h_ls) is the world’s most comprehensive and broadly adopted cloud platform, offering over 175 fully featured services from data centers globally. Millions of customers—including the fastest-growing startups, largest enterprises, and leading government agencies—are using AWS to lower costs, become more agile, and innovate faster.

## AWS Elastic Kubernetes Service

[AWS Elastic Kubernetes Services (EKS)](https://aws.amazon.com/eks/) is a fully managed [Kubernetes](https://kubernetes.io/) service provided by AWS. 

<br>

# Description of project

This tutorial will demonstrate how to create a **AWS Elastic Kubernetes Service (EKS)** managed cluster. This will be done in 3 mains steps :
- __Networking Setup__, in this section we will set up AWS network environment by creating different resources like a Virtual Private Cloud (VPC), Internet Gateway (IGW), Route Table, Routes, Subnets and Security Group (SG).

- __AWS EKS Cluster__, then in this section, we will create IAM Cluster (Control Plane) Role, provision EKS Control Plane, IAM Node Group Role and provision Node Group (Node Group is the name given by AWS to describe group of worker nodes).

- __Cluster Testing__, finally we will deploy a simple web application in the EKS Cluster to verify that it's working as expected.

<br>

# Tutorial Deployment 

To deploy the EKS Cluster we purpose two methods : 

-  [Infrastructure as Code (IaC)](https://github.com/samiamoura/aws-eks/tree/master/infrastructure-as-code) tutorial with AWS CloudFormation,
-  [AWS Command Line Interface](https://github.com/samiamoura/aws-eks/tree/master/aws-cli) tutorial with shell tool.

<br>

# Infrastructure 

## EKS Architecture for Control Plane and Worker Node Communication

![AWS schema communication](https://user-images.githubusercontent.com/58267422/99916723-10d23080-2d0c-11eb-851f-1d8dd8348bce.png)

<br>

## EKS Detailed Network Infrastructure

![AWS network schema](https://user-images.githubusercontent.com/58267422/99916762-4d059100-2d0c-11eb-9439-0a9dea74cdad.png)

<br>

## EKS Project Architecture

![AWS Schema infrastructure](https://user-images.githubusercontent.com/58267422/99916726-13348a80-2d0c-11eb-9e6a-f6a133d8d004.png)