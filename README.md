This project demonstrates how to fully configure AWS infrastructure using Terraform. It covers the complete process—from installing and configuring the AWS CLI, creating AWS access keys, installing Terraform, and running Terraform commands—to provision basic AWS resources such as a Virtual Private Cloud (VPC), subnets, an Internet Gateway, a security group, an EC2 instance, and an S3 bucket.

## Overview

This project uses Terraform to provision the following AWS resources:

- A custom VPC with a specified CIDR block.
- A public subnet in the default VPC.
- An Internet Gateway and associated route table for outbound Internet access.
- A security group that allows inbound SSH and HTTP access.
- An EC2 instance (using Amazon Linux 2) configured via a user data script.
- An S3 bucket for storage purposes.

The goal is to provide a fully automated and reproducible AWS infrastructure setup that can be used as a foundation for more advanced projects.

---

## Prerequisites

Before you begin, ensure that you have the following:

- **AWS Account:** Sign up for an AWS account if you do not already have one.
- **AWS CLI:** Installed on your local machine.
- **Terraform:** Installed on your local machine.
- **SSH Client:** To connect to the EC2 instance (typically built into Linux and macOS; Windows users may use PuTTY or Windows Terminal).

---

## Setup and Installation

### Installing AWS CLI

1. **For macOS/Linux:** 

- Download and install using the bundled installer:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

- Verify installation:

```bash
aws --version
```

2. **For Windows:**

- Download the MSI installer from the [AWS CLI website](https://aws.amazon.com/cli/) and follow the installation instructions.
- Verify installation using Command Prompt:

```bash
aws --version
```

### Creating AWS Access Keys

1. Sign in to the [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to **IAM (Identity and Access Management)**.
3. Create a new IAM user (or use an existing one) with programmatic access.
4. Attach a policy (for learning purposes, you can use the **AmazonEC2FullAccess** and **AmazonS3FullAccess** policies, but for production use always follow the principle of least privilege).
5. After creating the user, download or copy the **Access Key ID** and **Secret Access Key**.
6. Configure the AWS CLI with these keys:

```bash
aws configure
```

You will be prompted for:

- AWS Access Key ID
- AWS Secret Access Key
- Default region name (e.g., `us-east-1`)
- Default output format (e.g., `json`)

### Installing Terraform

1. **For macOS/Linux:**

- Download the latest version from Terraform's website.
- Unzip the package:

```bash
unzip terraform_1.X.X_linux_amd64.zip
```

 - Move the Terraform binary to a directory in your PATH, for example:

```bash
sudo mv terraform /usr/local/bin/
```

- Verify installation:

```bash
terraform version
```

2. **For Windows:**

- Download the Terraform zip from Terraform's website.
- Extract the executable and add it to your system PATH.
- Verify installation by opening Command Prompt:

```bash
terraform version
```


## Terraform Configuration

### Project Structure

A recommended project structure for this project is:

```bash
aws-terraform-project/
├── main.tf
├── variables.tf
├── terraform.tfvars  # (optional, for variable values)
```

### Terraform Code Overview

#### **main.tf**

This file contains the resource definitions for the VPC, subnet, Internet Gateway, route table, security group, EC2 instance, and S3 bucket.

```bash
provider "aws" {
  region = var.aws_region
}

# Use the default VPC in your account
data "aws_vpc" "default" {
  default = true
}

# Retrieve all subnets in the default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Create a security group that allows SSH and HTTP access
resource "aws_security_group" "infrastructure_sg" {
  name        = "infrastructure-sg"
  description = "Allow SSH (22) and HTTP (80) access"
  vpc_id      = data.aws_vpc.default.id

  ingress {
    description = "Allow SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "Allow HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Import an SSH key pair into AWS
resource "aws_key_pair" "my_key" {
  key_name   = var.key_name
  public_key = file(var.public_key_path)
}

# Create an EC2 instance running Amazon Linux 2 with a basic user_data script
resource "aws_instance" "app_instance" {
  ami                    = var.amazon_linux_ami
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnets.default.ids, 0)
  key_name               = aws_key_pair.my_key.key_name
  vpc_security_group_ids = [aws_security_group.infrastructure_sg.id]

  user_data = <<-EOF
    #!/bin/bash
    # Update the instance and install necessary packages
    yum update -y
    # (Additional commands can be added here)
  EOF

  tags = {
    Name = "AppInstance"
  }
}

# Create an S3 bucket (bucket ACLs are now enforced by default)
resource "aws_s3_bucket" "app_bucket" {
  bucket = var.bucket_name
}
```

#### **variables.tf**

This file defines the input variables used in the configuration.

```bash
variable "aws_region" {
  description = "AWS Region"
  default     = "us-east-1"
}

variable "amazon_linux_ami" {
  description = "Amazon Linux 2 AMI ID (free tier eligible)"
  default     = "ami-0c02fb55956c7d316"  # Ensure this AMI exists in your region
}

variable "key_name" {
  description = "Name for the EC2 Key Pair"
  default     = "my-key"
}

variable "public_key_path" {
  description = "Path to your public key file (e.g., my-key.pem.pub)"
  default     = "my-key.pem.pub"
}

variable "bucket_name" {
  description = "A globally unique name for the S3 bucket"
  default     = "my-unique-aws-bucket-123456"  # Change to a globally unique name
}
```

## Usage Instructions

1. **Configure AWS CLI:**  
    Run `aws configure` and provide your AWS Access Key ID, Secret Access Key, region, and output format.
    
2. **Initialize Terraform:**  
    Navigate to the project directory and run:

```bash
terraform init
```

3.  **Review the Terraform Plan:**  

```bash
terraform plan
```

This command shows you what Terraform plans to create.

4.  **Apply the Configuration:**  

```bash
terraform apply
```

- Confirm the prompt by typing `yes`. Terraform will create your infrastructure.
    
5. **Verify Your Resources:**
    
    - Check the AWS Console to see the newly created VPC, EC2 instance, and S3 bucket.
    - Use your SSH client to connect to the EC2 instance if needed:

```bash
ssh -i my-key.pem ec2-user@<EC2_PUBLIC_IP>
```


## Cleaning Up

To remove all the resources created by Terraform, run:

```bash
terraform destroy
```

Confirm by typing `yes` when prompted.

