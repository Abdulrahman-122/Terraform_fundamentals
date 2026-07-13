How to use Terraform  as  developer?
- write +  update terraform code.
- test locally  on the   environment
- create  a pull request in a version control.
- Have a colleague review your pull request
- run tests using CI on  github
- if the test  pass  -> merge test to the main  branch
- use automated pipeline  in the CI/CD to  deploy the changes to the staging environment
- deploy the changes  to production environment.
There  are a lot of third party  tools that help you to  run your terraform   like;
 - terragrunt
 - CI/CD tools(Github Actions,Gitlab...)
Some recommendations:
- Be cautious when changing resource names.
- Protect sensitive data in Terraform state files.
- Configure cloud timeouts to prevent partial provisioning.
- Avoid naming conflicts when provisioning resources.
- Remember to destroy test infrastructures.
- Ensure all team members use the same Terraform version.
- Understand that there are multiple ways to accomplish the same configuration.
- Be aware of immutable parameters in resources.
- Avoid making changes outside of the Terraform sequence.


# Your next step -> pick idea -> try to build it using a terraform 
- there are some ideas you can start with ;

Our goal is that after these labs, you'll be able to look at almost any Terraform file and explain what every resource is doing.

---

# Terraform Bootcamp (10 Labs)


```
Lab 1
      ↓
Lab 2
      ↓
Lab 3
      ↓
...
      ↓
Lab 10

Final Project
```

Each lab introduces only **one or two new ideas**.

---

# Lab 1 — Your First Infrastructure

### Goal

Create one EC2 instance.

Nothing else.

No security groups.

No variables.

No outputs.

No backend.

Just:

```
Terraform

↓

AWS

↓

EC2
```

---

## What you'll learn

```
provider

terraform block

resource

terraform init

terraform plan

terraform apply

terraform destroy
```

---

## Folder

```
Terraform/
    Lab01/
        main.tf
```

---

## Code

```hcl
terraform {
  required_version = ">= 1.15"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.92"
    }
  }
}

provider "aws" {
  region = "eu-west-3"
}

resource "aws_instance" "web" {

  ami           = "ami-0e1c4170d9c01184b"

  instance_type = "t3.micro"

}
```

---

## Commands

```
terraform init
```

↓

```
terraform plan
```

↓

```
terraform apply
```

↓

Go to EC2 Console.

See the instance.

↓

```
terraform destroy
```

---

## Questions you should answer

After this lab you should know

Why do we need

```
terraform
```

block? in order to provide our instances the providers 

---

Why do we need

```
provider
```

? in order to make the instances on their infrastructure

---

Why is

```
aws_instance
```

called a resource? as it create for us the instance in order to b

---

What does

```
"web"
```

mean? this is the name that terraform will be used while working on our insatnce

---

Why isn't Terraform asking AWS for the AMI automatically? as it needs to configure first the credentials of the user who want to run the instances this is good for security...
## Summary

|Question|Improved Answer|
|---|---|
|Why do we need the `terraform` block?|To configure Terraform itself, including required providers, Terraform version, and backend/state settings.|
|Why do we need the `provider` block?|To tell Terraform how to connect to a cloud provider and where to create resources.|
|Why is `aws_instance` called a resource?|Because it represents infrastructure that Terraform creates and manages—in this case, an EC2 instance.|
|What does `"web"` mean?|It's the local name Terraform uses to identify and reference this specific resource in the configuration.|

---

# Lab 2 — Outputs

Current infrastructure

```
EC2
```

Now we'll learn how Terraform gives us information back.

We'll add

```
output
```

Example

```
Public IP

Public DNS

Instance ID
```

---

You'll learn

```
aws_instance.web.id

aws_instance.web.public_ip

aws_instance.web.public_dns
```

---


---

# Lab 3 — Security Groups

Current

```
Internet

↓

EC2
```

We'll insert a firewall.

```
Internet

↓

Security Group

↓

EC2
```

Now you'll finally understand

```
ingress

egress

tcp

22

80

443

8080

CIDR
```

---

# Lab 4 — User Data

Now our EC2 becomes useful.

Terraform starts

```
Python HTTP Server
```

automatically.

When you open

```
http://<EC2-IP>:8080
```

You'll see

```
Hello AbdulRahman
```

This is your first real cloud application.

---

# Lab 5 — Multiple EC2

Instead of

```
1 EC2
```

Create

```
3 EC2
```

You'll learn

```
count

count.index

for_each
```

---

# Lab 6— S3

Now create

```
S3 Bucket

Versioning

Encryption
```

Exactly what you already practiced.

Now you'll actually understand why.

---

# Lab 7 — Remote Backend

Now

```
State

↓

S3

↓

Lock

↓

DynamoDB
```

You'll finally understand

```
terraform.tfstate

remote backend

state locking

bootstrap
```

---

# Lab 8— Load Balancer

Architecture

```
Internet

↓

ALB

↓

EC2

EC2
```

You'll understand

```
Listener

Listener Rule

Target Group

Health Check
```

---

# Lab 9 — Database

Create

```
PostgreSQL
```

using Terraform.

Learn

```
RDS

Endpoint

Subnet

Credentials

Security Group
```

---

# Final Project

Now combine everything.

```
Internet

↓

Route53

↓

Application Load Balancer

↓

Security Group

↓

EC2 #1

↓

EC2 #2

↓

RDS

↓

Remote Backend

↓

S3

↓

DynamoDB
```



    
