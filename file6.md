# Terraform_backend
1. ✅ Local backend (you've already done this)
    
2. ✅ **Terraform Cloud (HCP Terraform)** — easiest, completely free for personal use
    
3. ✅ **AWS S3 + DynamoDB** — what many AWS companies use
    
4. Later: GitHub Actions + remote backend
    

Let's do it as a lab.

---

# What is a Terraform backend?

When you run

```bash
terraform apply
```

Terraform creates

```
terraform.tfstate
```

This file contains everything Terraform knows about your infrastructure.

Example:

```
terraform.tfstate

EC2
 ├── id=i-0ab12...
 ├── public_ip
 ├── subnet
 ├── security group
 └── ...
```

Terraform **doesn't ask AWS every time**.

Instead it compares

```
Current infrastructure
        VS
terraform.tfstate
        VS
main.tf
```

to determine what to create, update, or destroy.

---

# Why is local state bad?

Suppose you and I work together.

You have

```
terraform.tfstate
```

on your laptop.

I have another copy.

Now both of us execute

```
terraform apply
```

Terraform has two different "truths."

Eventually one of us overwrites the other.

This is why remote state exists.

---

# Option 1 — HCP Terraform (Terraform Cloud)

This is what I recommend **first**.

Advantages:

- completely free for individuals
    
- state stored securely
    
- encrypted
    
- version history
    
- state locking built in
    
- no AWS resources needed
    

This is what many companies use.

---

# Step 1

Go to

> [https://app.terraform.io](https://app.terraform.io)

Create a free account.

---

# Step 2

Create an Organization.

Example

```
Hohn-org
```

---

# Step 3

Create a Workspace.

Example

```
terraform-lab
```

Choose

```
CLI Driven Workflow
```

not GitHub.

---

# Step 4

Terraform Cloud shows something similar to

```hcl
terraform {

  cloud {

    organization = "abdo-org"

    workspaces {
      name = "terraform-lab"
    }

  }

}
```

Copy it.

---

# Step 5

Inside your project

```
main.tf
```

put

```hcl
terraform {

  cloud {

    organization = "abdo-org"

    workspaces {
      name = "terraform-lab"
    }

  }

  required_providers {

    aws = {

      source = "hashicorp/aws"

      version = "~>5.92"

    }

  }

}
```

Notice

No backend block.

The

```
cloud {}
```

block replaces it.

---

# Step 6

Login

Run

```bash
terraform login
```

Browser opens.

Approve.

Terraform stores a token in

```
~/.terraform.d/credentials.tfrc.json
```

Now you're authenticated.

if you got that problem;
terraform init  
╷  
│ Error: Unsupported Terraform Core version  
│    
│   on run_project.tf line 2, in terraform:  
│    2:   required_version = "1.15.7"  
│    
│ This configuration does not support Terraform version 1.15.6. To proceed, either choose  
│ another supported Terraform version or update this version constraint. Version constraints are  
│ normally set for good reason, so updating the constraint may lead to other errors or  
│ unexpected behavior.  
╵
then install tfenv:
On Arch:

```
yay -S tfenv
```

Then:

```
tfenv install latesttfenv use latest
```

Or install a specific version:

```
tfenv install 1.15.7tfenv use 1.15.7
```

Now:

```
terraform version
```

should show the selected version.

then if you have multi files that end with .tf 
you need to just handle one in the current time in order to make it clear for terraform;
so ;
move every file to .bak in order to make it track the new one
```
mv  basic.tf basic.tf.bak 
```
if you didn't make this it will give you error

---

# Step 7

Initialize

```bash
terraform init
```

Terraform asks

```
Do you want to copy existing state?
```

Answer

```
yes
```

Now

```
terraform.tfstate
```

is uploaded.

Initializing provider plugins found in the configuration...  
- Reusing previous version of hashicorp/aws from the dependency lock file  
- Using previously-installed hashicorp/aws v5.100.0  
  
Initializing HCP Terraform...  
  
  
  
HCP Terraform has been successfully initialized!  
  
You may now begin working with HCP Terraform. Try running "terraform plan" to  
see any changes that are required for your infrastructure.  
  
If you ever set or change modules or Terraform Settings, run "terraform init"  
again to reinitialize your working directory.  


---

# Step 8

Verify

Open

```
Workspace

↓

States
```

You'll see

```
terraform.tfstate
```

stored online.

Congratulations.

You're now using Terraform Cloud.

---

# How locking works

Imagine

You

```
terraform apply
```

While it's running

Your teammate also runs

```
terraform apply
```

Terraform Cloud says

```
Workspace Locked
```

Second person waits.

Nobody corrupts the state.

---

# Option 2 — AWS S3 Backend

Now let's build it ourselves.

This teaches how remote state actually works.

---

# Architecture

```
                Terraform

                    │

                    ▼

              S3 Bucket
         terraform.tfstate

                    │

                    ▼

            DynamoDB Table

              LockID
```

S3 stores

```
terraform.tfstate
```

DynamoDB stores

```
LockID
```

Only one Terraform process can own the lock.

---

# Why DynamoDB?

Without DynamoDB

Developer A

```
terraform apply
```

Developer B

```
terraform apply
```

Both update the state simultaneously.

Eventually corruption.

DynamoDB prevents this.

---

# Bootstrapping Problem

Question:

How do we create the S3 bucket that Terraform itself needs?

Answer:

Temporarily use

```
Local backend
```

Create

```
S3
```

and

```
DynamoDB

```

Then migrate.

This is called

```
Bootstrap
```

---

# bootstrap/

Create

```
bootstrap/

main.tf
```

Inside

```hcl
provider "aws" {

  region="eu-west-3"

}
```

Create bucket

```hcl
resource "aws_s3_bucket" "state" {

  bucket="abdo-terraform-state"

}
```

Bucket names must be globally unique, so use something like:

```
abdo-terraform-state-344859353091
```

(using your AWS account ID helps ensure uniqueness).

---

Enable versioning

```hcl
resource "aws_s3_bucket_versioning" "versioning" {

  bucket=aws_s3_bucket.state.id

  versioning_configuration {

      status="Enabled"

  }

}
```

---

Enable encryption

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "crypto" {

 bucket=aws_s3_bucket.state.bucket

 rule{

   apply_server_side_encryption_by_default{

      sse_algorithm="AES256"

   }

 }

}
```

---

Create DynamoDB

```hcl
resource "aws_dynamodb_table" "locks" {

 name="terraform-locks"

 billing_mode="PAY_PER_REQUEST"

 hash_key="LockID"

 attribute{

   name="LockID"

   type="S"

 }

}
```

---

Apply

```bash
terraform init

terraform apply
```

Now AWS contains

```
S3 Bucket

+

DynamoDB Table
```

---

# Now switch backend

Add

```hcl
terraform {

 backend "s3" {

   bucket="abdo-terraform-state-344859353091"

   key="dev/terraform.tfstate"

   region="eu-west-3"

   dynamodb_table="terraform-locks"

   encrypt=true

 }

}
```

Run

```bash
terraform init
```

Terraform asks

```
Copy existing state?

```

Type

```
yes
```

Done.

Your

```
terraform.tfstate
```

is now stored inside S3.

---

# Verify

Open AWS

S3

↓

Your bucket

↓

You'll see

```
terraform.tfstate
```

Open DynamoDB

↓

terraform-locks

When apply is running

You'll briefly see

```
LockID
```

appear.

---

# Which backend should you use?

Since your goal is becoming a DevOps/SRE engineer, here's what I'd recommend:

- **Learn first:** HCP Terraform (Terraform Cloud). It's the easiest way to understand remote state, state history, and locking without managing AWS infrastructure.
    
- **Then learn:** AWS S3 + DynamoDB. This teaches how a self-managed backend works and is still common in AWS-focused organizations.
    
- **Finally:** Integrate either backend into your GitHub Actions CI/CD pipelines so your infrastructure deployments use the same shared state.
    
