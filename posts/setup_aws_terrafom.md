---
author : "Cesar Sepulveda"
categories : ["IaC"]
date : "2021-06-22T13:09:24Z"
description : "Npm install command help to install package from npmjs.org"
image : "images/post1/iac_cicd.png"
images : ["images/post1/iac_cicd.png"]
slug : "Initial-Setup-of-Terraform-and-GitHub-Actions"
summary : "Setup Terraform cloud and GitHub Acctions to deploy resources on AWS"
tags : ["terraform, aws, github, iac, ci/cd"]
title : "Initial Setup of Terraform and GitHub Actions"
draft : false
---

Terraform is an open-source infrastructure as a code software tool that enables you to safely and predictably create, change, and improve infrastructure.

Here I will describe how to integrate Terraform Cloud and GitHub Actions to deploy resources on AWS.
In that way, we could have Continuous Delivery and Continuous Delivery (CI/CD) in our Infrastructure as a Code (IaC)

![iac_cicd](/images/post1/iac_cicd.png "iac_cicd")

Some Stuff that we are going to need to realize this setup:

* Have an AWS account with access to create IAM users and grant it full admin privileges.
* Have a Terraform Cloud account.
* Have a GitHub account.

# Base Setup:
First, we have to create an IAM user and grant him Administrator Access.

![iam_account](/images/post1/iam_account.png "Create Account")
![iam_access](/images/post1/iam_access.png "Grant Access")

In a new browser tab, create a new organization in Terraform cloud, then its necessary to create a workspace using the `API-diven workflow`

![terraform_setup](/images/post1/terraform_setup.png "terraform_setup")

Then we have to create the Environment variables to access the AWS resources; these values must be added as `sensitive` values.
Also, we are going to create the Terraform variable `region` to specify which region of AWS we are going to use by default.

![terraform_variables](/images/post1/terraform_variables.png "terraform_variables")

Now we are going to create a new GitHub repository and upload the initial files to get ready our setup

![github_setup](/images/post1/github_setup.png "github_setup")

Now we must back to terraform to create a `Team API Token` and set up this in our Github Repo as a Secret; this will allow to Github Actions use our Terraform Workspace

![terraform_token](/images/post1/terraform_token.png "terraform_token")

Now we are going to create a new branch and upload the essential files to this new branch.
When the open a new Pull Request, GitHub action will run, and when a merge being done against the main branch, Github Actions will execute the terraform Apply.

![github_actions](/images/post1/github_actions.png "github_actions")

The full code could be checked here:
[https://github.com/csepulveda/aws_base_setup/pull/1/files](https://github.com/csepulveda/aws_base_setup/pull/1/files)

The repo structure is the following:

```
.github/
└── workflows
    └── terraform.yml  #This cointains the GitHub actions code.
.
├── main.tf            #terraform backend remote configuration
├── providers.tf       #instance the aws provider
├── README.md          #sample readme file
└── variables.tf       #default variables
```

When the files are uploaded, and we create a new PR, we could see the GitHub actions working.

![branch](/images/post1/branch.png "branch")

If everything goes fine, we could see this final status in our GitHub actions:

![final](/images/post1/final.png "final")

Now it is possible to start creating resources using this repository. There is no need to make any terraform action from our local computer or be worried about where to store our terraform states.

Also, anyone with access to the repo could add new resources without any access to any sensitive value.