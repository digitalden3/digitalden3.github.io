---
layout: post
title: "Creating and Deploying Reusable Amazon VPC & NAT Gateway Modules with Terraform"
date: 2023-08-11 08:00:00 - 0500
categories: aws terraform
tags: vpc
image:
  path: /assets/images/terraform_modules.webp
---

In Terraform, modules serve as fundamental tools for structuring infrastructure code. They enable the encapsulation and reuse of specific sections of code, promoting cleanliness and modularity within configurations. Modules can be likened to templates, defining resources or sets of resources and facilitating the management of complex infrastructures by breaking them down into manageable components.

Today, we are going to take a look at the practical aspects of Terraform modules, demonstrating how they simplify the process of establishing and configuring your Virtual Private Cloud (VPC), nat gateways, and related resources. The objective is to make your infrastructure code clean, reusable, and easy to maintain.

The documentation encompasses the entire process, starting from setting up the Terraform project structure in Visual Studio Code, configuring a remote backend on AWS with S3 and DynamoDB, creating reusable modules for VPC and nat gateways, defining Terraform variables, and overseeing the complete deployment process.

Please note that all the Terraform code featured in this article has already been written for your convenience. However, if you prefer to build the Terraform modules completely from scratch, please follow along with the in-depth video demonstration below.

{% include embed/youtube.html id='tb7a0QXUaOM' %}
📺 [Watch Demo](https://www.youtube.com/watch?v=tb7a0QXUaOM)

## Overview

Explore the process of creating and deploying a reusable Amazon VPC and nat gateway infrastructure with Terraform modules.

1. `Preparing Your AWS Environment`
   Before starting with Terraform, you will ensure a streamlined and secure workflow by setting up your AWS environment and project structure. This includes verifying the installation of the AWS CLI and Terraform, creating a well-organised project folder structure, and configuring essential Terraform files.

2. `Configuring Terraform Backend Using S3 and DynamoDB with State Locking`
   To enhance consistency and security, you will configure Terraform to use S3 for storing state files and DynamoDB for state locking. This critical step ensures smooth collaboration and management in your infrastructure project.

3. `Creating a Reusable VPC Module`
   You will create a reusable VPC module in Terraform to promote best practices and simplify infrastructure management. This module, organised into main, variables, and outputs files, will handle the creation of your VPC, internet gateway, availability zones, subnets, and route tables.

4. `Defining Terraform Variables and Main Configuration for VPC Module Deployment`
   Simplify VPC setup by defining Terraform variables and configuring the main Terraform configuration. Create variables like AWS region, project name, VPC CIDR block, and subnet CIDR blocks in variables.tf. Employ a terraform.tfvars file for easy configuration adjustments. Configure main.tf to establish a secure connection to AWS using the AWS provider, deploying the VPC module with specified variables for streamlined management.

5. `Initializing Terraform and Applying Configuration for VPC Deployment`
   You will initializing and apply your Terraform configuration to create the VPC and related resources. This modular approach enhances reusability and maintainability while simplifying infrastructure management.

6. `Creating And Deploying A Reusable NAT Gateway Module`
   For enhanced security and availability, you will create and deploy a reusable NAT gateway module in Terraform. This module, complete with main and variables files, will automate the setup of NAT gateways, Elastic IPs, and route tables.

7. `Destroying Resources with Terraform`
   Lastly, you will dismantle your AWS infrastructure, including VPC and NAT Gateway modules, using Terraform’s destroy command. This ensures that all resources are cleanly and securely removed when they’re no longer needed.

### Directory Structure

To start lets go over the directory structure you will establish by the end of the Terraform project

![img-description](/assets/images/directory_structure.webp)
_Tree Command_

1. `Root Directory:` You will begin by creating a root directory for your Terraform project
2. `AWS-VPC Directory:` Inside the root directory, you will create an AWS-VPC directory for your main Terraform project. Here, you will include the configuration files for your AWS VPC setup
3. `Modules Directory:` At the root level, you will create a modules directory to store modularized Terraform configurations.
4. `NAT-Gateway Module:` Inside the modules directory, you will create a NAT-gateway directory to define a module for NAT gateway configuration
5. `VPC Module:` Within the Modules directory, you will establish a VPC directory for a module dedicated to VPC configuration
6. `Organize Files:` You will populate each directory with relevant Terraform files, such as main.tf, variables.tf, outputs.tf, terraform.tfvars, and backend.tf, to define and organize your infrastructure-as-code project

This structured approach facilitates the organization and modularization of your Terraform project, enhancing manageability and maintainability as your infrastructure requirements evolve.

### Architecture

The architecture you are going to set up provides a multi-AZ VPC with public and private subnets in the us-east-1 region. Instances in the public subnets can access the internet, while instances in the private subnets can route outbound traffic through nat gateways for internet access while remaining protected from incoming internet traffic.

![img-description](/assets/images/architecture1.webp)

`Subnets:` Two public subnets with CIDR blocks 10.0.0.0/24 and 10.0.1.0/24, respectively, are created and associated with the VPC. Additionally, two private subnets with CIDR blocks 10.0.2.0/24 and 10.0.3.0/24, respectively, are created and associated with the VPC. The public subnets are configured to map public IP addresses to instances.

`Public Route Table:` A public route table is created and associated with the VPC, enabling public internet access for resources in the public subnets. It routes traffic through the Internet Gateway.

`Route Table Associations:` The public subnets are associated with the public route table.

`Elastic IPs (EIPs):` Elastic IPs are allocated for use with NAT gateways in both Availability Zones to facilitate outbound internet access for resources in the private subnets.

`NAT Gateways:` NAT gateways are created in the public subnets and associated with the allocated Elastic IPs. They allow private instances in the associated private subnets to initiate outbound traffic to the internet while blocking incoming traffic from the internet.

`Private Route Tables:` Private route tables are created and associated with the VPC. Default routes in these tables point to the respective NAT gateway, allowing private instances to route outbound traffic through the NAT Gateway for internet access.

`Route Table Associations (Private):` The private subnets are associated with their respective private route tables.

To achieve this architecture, you will utilize Terraform modules, including a VPC module and a NAT gateway module, to automate the deployment. The code will be written to specify the desired configuration and resources for this architecture.

### Prerequisites

To follow this tutorial you will need:

- The Terraform CLI installed
- The AWS CLI is installed
- AWS account and Associated Credentials

## Task 1: Preparing Your AWS Environment

Before you start, ensure you have the AWS Command Lin Interface (CLI) installed and configured with the necessary access credentials. You should also have Terraform installed on your local machine.

You can check if you have the AWS CLI and Terraform installed on your terminal by running a simple command for each.

- Open Visual Studio Code (VS Code) on your computer.
- In the top navigation pane, choose Terminal, and in the dropdown, select New Terminal.
- In the Terminal, run the following commands to confirm the versions of AWS CLI and Terraform:

```shell
aws --version
terraform --version
```

If you see version information for both AWS CLI and Terraform, it means they are installed and available in your terminal. If you get errors or no output, it indicates that they are not installed, and you will need to download and install them to use them in your terminal.

![img-description](/assets/images/shell1.webp)

### a) Creating The Project Folder

Begin by creating a dedicated project folder. This is an essential step in organizing and structuring your Terraform project.

- In the Terminal run the make directory command to create a new project folder:

```shell
mkdir Terraform-AWS-VPC
```

Next, create another folder for your specific project in your projects root directory.

- In the Terminal run the make directory command to create a specific project folder:

```shell
mkdir mkdir AWS-VPC
```

You directory should now look like this:

![Alt text](/assets/images/shell2.webp)

### b) Creating Main, Versions, Variables, And Terraform.tfvars Files

In AWS-VPC folder you will create main, versions, variables, and Terraform.tfvars files. Overall, this structure helps you write more modular and maintainable Terraform code, making it easier to manage complex infrastructure configurations across different environments or projects.

`main.tf:` This file typically contains the primary configuration for your infrastructure. It defines the resources, modules, providers, and other settings needed to create and manage your infrastructure components. In essence, main.tf is where you specify what resources you want to create and how they should be configured.

- With AWS-VPC selected, click on the New File icon, type main.tf, and press Enter

`versions.tf:` Allows you to specify the minimum required version of Terraform for your project. This helps ensure that users of your configuration are using a compatible Terraform version.

- With AWS-VPC selected, click on the New File icon, type versions.tf, and press Enter

In general, for larger, team-based, or long-term projects, specifying versions is advisable because it helps maintain stability and consistency. It also provides documentation for project users and contributors. However, for small, short-term projects, it may be less critical.

`variables.tf:` This file is used to declare input variables that allow you to parameterize your Terraform configurations. Instead of hardcoding values directly into your main.tf file, you can define variables in variables.tf and then reference these variables in your main configuration. This makes configurations more flexible and reusable since you can provide different values for these variables depending on the environment or use case.

- With AWS-VPC selected, click on the New File icon, type variables.tf, and press Enter

`terraform.tfvars:` This file is where you assign values to the variables declared in variables.tf. It is essentially a configuration file where you set the actual values for variables used in your Terraform configuration. By separating the variable definitions (variables.tf.) from the values (terraform.tfvars), you can keep sensitive information (like API keys or secrets) out of your main configuration and safely manage them separately.

- With AWS-VPC selected, click on the New File icon, type terraform.tfvars, and press Enter.

![Alt text](/assets/images/shell3.webp)

### c) Configuring A Dedicated AWS CLI Profile for Terraform

By using a dedicated profile for Terraform, you gain greater control, security, and separation of responsibilities in your AWS interactions, which is considered a best practice for managing infrastructure with Terraform.

You can create an AWS CLI profile using the aws configure command with the profile flag followed by the profile name.

- In the Terminal run the following command to configure your AWS CLI profile:

```shell
aws configure --profile terraform-user
```

You will be prompted to enter your AWS Access Key ID, AWS Secret Access Key, default region name, and default output format:

- `AWS Access Key ID:` Enter your Access Key ID.
- `AWS Secret Access Key:` Enter your Secret Access Key
- `Default region name:` Enter us-east-1.
- `Default output format:` You can leave this blank or enter json for JSON output.

Once you have entered this information, the AWS CLI will save it under the profile name you specified.

- You can view and edit your AWS CLI configuration by running:

```shell
aws configure list-profiles
```

![Alt text](/assets/images/shell4.webp)

You can now use the terraform-user profile when running AWS CLI commands by specifying the profile flag followed by the profile name. For example:

```shell
aws s3 ls --profile terraform-user
```

This profile will be used for authentication and authorization when interacting with AWS services via the AWS CLI.

By following these steps, you have created a dedicated AWS CLI profile for Terraform, allowing you to manage your infrastructure with isolated and secure credentials.

## Task 2: Configuring Terraform Backend Using S3 and DynamoDB With State Locking

Terraform state (often referred to as tfstate), refers to the JSON file that Terraform maintains to store the current state of your infrastructure. This file is named terraform.tfstate by default, but you can configure Terraform to use a different name and location if needed.

### a) Configuring an S3 Bucket for Storing Terraform State

Creating an S3 bucket to store Terraform state ensures proper management, coordination, and tracking of your infrastructure’s current state. It is a best practice for working with Terraform in collaborative or production environments to avoid conflicts and maintain consistency.

- Use the AWS CLI to create an S3 bucket that will store your Terraform state files:

```shell
aws s3api create-bucket \
  --bucket digitalden-terraform-tfstate \
  --profile terraform-user
```

Replace digitalden-terraform-tfstate with your desired, globally unique bucket name.

![Alt text](/assets/images/shell5.webp)

This setup is a crucial step in maintaining the integrity and reliability of your Terraform deployments.

### b) Setting Up A DynamoDB Table for State Locking

Creating a DynamoDB table for state locking is a best practice when working with Terraform, especially in collaborative or production environments. It adds a layer of control and coordination to Terraform operations, enhancing consistency, security, and overall reliability in managing your infrastructure as code.

To create a DynamoDB table specifically for state locking. Run the aws dynamodb create-table command:

````shell
aws dynamodb create-table \
    --table-name terraform-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
    --profile terraform-user
    ```shell
````

- `table-name:` Specifies the name of the DynamoDB table as terraform-locks. You can use a different name if desired
- `attribute-definitions:` Defines the attributes of the DynamoDB table. In this case, you have an attribute named LockID with a type of string (S)
- `key-schema:` Specifies the primary key of the table, which is LockID
- `provisioned-throughput:` Sets the read and write capacity units for the DynamoDB table
- `profile:` Ensures that the terraform-user AWS CLI profile is used for this specific command
