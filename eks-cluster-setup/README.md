# AWS EKS Cluster Setup Guide

This document explains how to create and configure an Amazon EKS cluster 
for deploying the Kubernetes Observability Demo project.

---

## 📌 Overview

We will create:

- An Amazon EKS cluster
- Managed node group
- IAM role configuration
- kubectl access configuration

Cluster will be created in:

Region: ap-south-1

---

## 🧰 Prerequisites

Make sure the following tools are installed:

- AWS CLI (configured with IAM permissions)
- eksctl
- kubectl
- Helm

Verify installations:

aws --version  
eksctl version  
kubectl version --client  

---

## 🌩 Step 1: Create EKS Cluster

Create cluster using eksctl:
 
```bash
eksctl create cluster \
--name=observability-cluster \
--region=ap-south-1 \
--zones=ap-south-1a,ap-south-1b \
--without-nodegroup
```
* Creates a EKS cluster with name observability-cluster in ap-south-1 region without node group.

##  step 2: Create and link OIDC Identity provider 

```bash
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=observability-cluster \
  --approve
```
* Creates an OIDC identity provider in AWS IAM and links it to our EKS cluster in ap-south-a region.

## step 3: Create node group to our cluster

```bash
eksctl create nodegroup \
  --cluster=observability-cluster \
  --region=ap-south-1 \
  --name=observability-nodes \
  --node-type=t3.medium \
  --nodes-min=2 \
  --nodes-max=3 \
  --node-volume-size=20 \
  --managed \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access \
  --node-private-networking
```
* creates a worker node group with name "observability-nodes" for our existing "observability-cluster".
* It provisions EC2 instances of type t3.medium with a minimum of 2 nodes and allows scaling up to 3 nodes using an Auto Scaling Group, each node having a 20 GB EBS volume for storage.
* The --managed flag ensures AWS manages lifecycle tasks like upgrades and patching.

## step 4: updates our local kubeconfig file

```bash
aws eks update-kubeconfig --name observability-cluster
```
* Updates our local kubeconfig file (usually located at ~/.kube/config) so that kubectl can connect to your Amazon EKS cluster named "observability-cluster".
* In simple terms, this command tells our local Kubernetes CLI how to authenticate and communicate with your EKS cluster in AWS.

## step 4: Verify Cluster

```bash
kubectl get nodes
```

# Install AWS Load Balancer Controller

## step 1: Associate IAM OIDC provider (we already did in step2 of cluster setup)

## step 2: Create IAM policy

* Download IAM policy JSON from AWS documentation and create policy:

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json 
```
## step 3: Create a IAM Service account

```bash
eksctl create iamserviceaccount \
--cluster=observability-cluster \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```
* This binds:
  * k8s service account
  * AWS IAM role
  * IAM Policy

## step 4: Install AWS Load Balancer Controller using helm

```bash
helm repo add eks https://aws.github.io/eks-charts
```
* This will adds the official Amazon EKS Helm chart repository to your local Helm configuration and gives it the name eks

```bash
helm repo update
```
* Helm to fetch the latest chart index files from all the Helm repositories you previously added

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
--set clusterName=observability-cluster \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```
* This will installs the AWS Load Balancer Controller Helm chart from the eks Helm repository into your EKS cluster.

## step 5: Verify controller is running

```bash
kubectl get pods -n kube-system | grep aws-load-balancer
```
* This is used to check whether the AWS Load Balancer Controller pods are running inside your cluster.
