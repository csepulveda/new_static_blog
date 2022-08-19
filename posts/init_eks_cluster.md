---
author : "Cesar Sepulveda"
categories : ["IaC"]
date : "2021-06-24T13:09:24Z"
description : "Npm install command help to install package from npmjs.org"
image : "images/post2/terraform.png"
images : ["images/post2/terraform.png"]
slug : "Setting_up_a_Simple_EKS_cluster_Using_Terraform_Github"
summary : "Setting up a Simple EKS with Autoscaling"
tags : ["terraform", "aws", "github", "iac", "ci/cd", "kubernetes"]
title : "Setting up a Simple EKS cluster Using Terraform/Github"
draft : false
---

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it easy for you to run Kubernetes on AWS and on-premises.

Here I will describe how to extend our [basic setup](/Initial-Setup-of-Terraform-and-GitHub-Actions.html) and setup a simple kubernetes cluster using EKS using the AWS/EKS terraform module and the Helm chart to install the Auto Scaling controller.

## Setup the VPC
The first step will be to set up our VPC; for this case, I'm going to create a VPC with three different subnets

* private_subnets, in this subnet, we are going to launch our node instances
* public_subnets, in this subnet, we are going to launch our load balancers (will not be used for the moment)
* database_subnets, in this subnet, we are going to launch our databases (will not be used for the moment)

The code could be checked [here](https://raw.githubusercontent.com/csepulveda/aws_base_setup/5343af5e3d852a4b005fbdc9bdd75e2b943bce8d/vpc.tf):

```
data "aws_availability_zones" "available" {
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "2.66.0"

  name                 = "k8s-vpc"
  cidr                 = "172.16.0.0/16"
  azs                  = data.aws_availability_zones.available.names
  private_subnets      = ["172.16.1.0/24", "172.16.2.0/24", "172.16.3.0/24"]
  public_subnets       = ["172.16.4.0/24", "172.16.5.0/24", "172.16.6.0/24"]
  database_subnets     = ["172.16.7.0/24", "172.16.8.0/24", "172.16.9.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  tags = {
    "kubernetes.io/cluster/${var.eks_cluster_name}" = "shared"
  }

  public_subnet_tags = {
    "kubernetes.io/cluster/${var.eks_cluster_name}" = "shared"
    "kubernetes.io/role/elb"                        = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/cluster/${var.eks_cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"               = "1"
  }
}
```

The tags will be used in the future to launch some Application Load Balancers using the AWS Ingress Controller.

## Setup the EKS Cluster
To setup the cluster, we are going to use the AWS/EKS module. It's pretty simple, and we have to modify a few things to enable the Auto Scaling controller.

The configuration is the [following](https://raw.githubusercontent.com/csepulveda/aws_base_setup/5343af5e3d852a4b005fbdc9bdd75e2b943bce8d/eks.tf):

```
data "aws_eks_cluster" "cluster" {
  name = module.eks.cluster_id
}

data "aws_eks_cluster_auth" "cluster" {
  name = module.eks.cluster_id
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.cluster.token
}

module "eks" {
  source = "terraform-aws-modules/eks/aws"

  cluster_name    = var.eks_cluster_name
  cluster_version = "1.20"
  subnets         = module.vpc.private_subnets
  enable_irsa     = true

  vpc_id = module.vpc.vpc_id

  worker_groups = [
    {
      instance_type = "t3.medium"
      asg_max_size  = 10 #this must be 10, but now its 0 to shutdown the instances
      asg_min_size  = 1  #this must be 1, but now its 0 to shutdown the instances
      tags = [
        {
          key                 = "k8s.io/cluster-autoscaler/enabled"
          value               = "TRUE"
          propagate_at_launch = true
        },
        {
          key                 = "k8s.io/cluster-autoscaler/${var.eks_cluster_name}"
          value               = "owned"
          propagate_at_launch = true
        }
      ]
  }]

  write_kubeconfig = false
}

output "kubectl" {
  value = "To craete your kubeconfig you must run:\n aws eks --region ${var.region} update-kubeconfig --name ${var.eks_cluster_name}"
}
```

Only with that Terraform will be able to create our EKS cluster.

To add the auto-scaling controller, we need to:

* Create an IAM role to allow the scaler to modify AWS resources
* Indicate to the ServiceRole which AWS IAM role must be used
* Install the controller.

To install the controller and set the permission, we are going to use the file: [eks_autoscaler.tf](https://raw.githubusercontent.com/csepulveda/aws_base_setup/41e3f622c9a55b4117dc33e1d65d83c229ac525a/eks_autoscaler.tf)

```
#To allow autoscaling we are going to create five differente resources:

#First resouce: AWS Autoscaler policy
#this will allow pods make calls to AWS Autoscaling resources
resource "aws_iam_policy" "cluster-autoscaler" {
  name        = "EKS-Cluster-Autoscaler"
  description = "Allow Pods use autoscaling resources."

  policy = file("iam-policies/autoscaler.json")
}

#Second resouce: AWS Trust relationships
#this will to some specifc serviceAccount in our k8s cluster use our IAM Role.
data "aws_iam_policy_document" "autoscaler_assume_policy" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    condition {
      test     = "StringEquals"
      variable = "${replace(data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer, "https://", "")}:sub"

      values = [
        "system:serviceaccount:kube-system:${var.eks_autoscaling_role_name}"
      ]
    }

    principals {
      type        = "Federated"
      identifiers = [module.eks.oidc_provider_arn]
    }
  }
}

#Third resouce: AWS Role
#this will create a role and attach our previous IAM Policy and Trust relationships
resource "aws_iam_role" "autoscaler_role" {
  name                = var.eks_autoscaling_role_name
  path                = "/"
  assume_role_policy  = data.aws_iam_policy_document.autoscaler_assume_policy.json
  managed_policy_arns = [aws_iam_policy.cluster-autoscaler.arn]
}

#Fourth resource: k8s cluster-autoscaler
#this helm chart will install the cluster-autoscaler service
resource "helm_release" "cluster-autoscaler" {
  name       = "cluster-autoscaler"
  chart      = "cluster-autoscaler"
  repository = "https://kubernetes.github.io/autoscaler"
  namespace  = "kube-system"

  set {
    name  = "autoDiscovery.clusterName"
    value = var.eks_cluster_name
  }
  set {
    name  = "awsRegion"
    value = var.region
  }
  set {
    name  = "rbac.serviceAccount.name"
    value = var.eks_autoscaling_role_name
  }
  set {
    name  = "rbac.serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.autoscaler_role.arn
  }
}
```
The installation of the controller will be done using HELM, and we have to pass a few arguments.

* autoDiscovery.clusterName:  To enable the autodiscovery of the resources that must be modified to apply the scale of services (the tags defined in the eks.tf file)
* awsRegion: The region name where our cluster will run
* rbac.serviceAccount.name: This is very important because will define with role must be attached to the serviceAccount used by the controller and grant access to the [resources](https://raw.githubusercontent.com/csepulveda/aws_base_setup/41e3f622c9a55b4117dc33e1d65d83c229ac525a/iam-policies/autoscaler.json)

Now, when we upload these files and run out a Pull Request as I already describe [here](/Initial-Setup-of-Terraform-and-GitHub-Actions.htm), we will have configurated our EKS cluster with auto-scaling.

## After applying the resources
We have to check if our cluster is doing the autoscaling correctly. To check that, we have to first check the logs from our autoscaling pods:

First, we need to create out Kube config, as shown out terraform output, for my case:
```
aws eks --region us-west-2 update-kubeconfig --name cluster1

```

Now we check the logs:
```
○ → kubectl -n kube-system logs -l app.kubernetes.io/name=aws-cluster-autoscaler -f


I0624 20:30:38.982046       1 filter_out_schedulable.go:132] Filtered out 0 pods using hints
I0624 20:30:38.982054       1 filter_out_schedulable.go:170] 0 pods were kept as unschedulable based on caching
I0624 20:30:38.982060       1 filter_out_schedulable.go:171] 0 pods marked as unschedulable can be scheduled.
I0624 20:30:38.982082       1 filter_out_schedulable.go:82] No schedulable pods
I0624 20:30:38.982116       1 static_autoscaler.go:402] No unschedulable pods
I0624 20:30:38.982136       1 static_autoscaler.go:449] Calculating unneeded nodes
I0624 20:30:38.982149       1 pre_filtering_processor.go:66] Skipping ip-172-16-3-243.us-west-2.compute.internal - node group min size reached

```
Check how many nodes we have
```
○ → kubectl get nodes
NAME                                         STATUS   ROLES    AGE   VERSION
ip-172-16-3-243.us-west-2.compute.internal   Ready    <none>   9h    v1.20.4-eks-6b7464
```

Then we could create a testing app and scale it to 50 instances pods

```
 2021-06-24 16:33:45 ⌚  csepulveda-Galago-Pro in ~
○ → kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created

 2021-06-24 16:33:49 ⌚  csepulveda-Galago-Pro in ~
○ → kubectl scale deployments/hello-node --replicas=50
deployment.apps/hello-node scaled

```

Then if we check it again, we are going to have more nodes:
```
○ → kubectl get nodes
NAME                                         STATUS     ROLES    AGE   VERSION
ip-172-16-1-54.us-west-2.compute.internal    NotReady   <none>   9s    v1.20.4-eks-6b7464
ip-172-16-2-190.us-west-2.compute.internal   NotReady   <none>   8s    v1.20.4-eks-6b7464
ip-172-16-3-243.us-west-2.compute.internal   Ready      <none>   9h    v1.20.4-eks-6b7464
ip-172-16-3-99.us-west-2.compute.internal    NotReady   <none>   4s    v1.20.4-eks-6b7464
```