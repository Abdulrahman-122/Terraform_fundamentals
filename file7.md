# build lab using this infrastructure
<img width="434" height="700" alt="image" src="https://github.com/user-attachments/assets/6401cdb6-7cff-465d-bd60-7b882e15fddd" />
this is a link to my lab that i built :https://github.com/Abdulrahman-122/Terraform_Project1
now let's build this simple project:

---

The backend configuration goes within the top level `terraform {}` block.

```json
terraform {
  # Assumes s3 bucket and dynamo DB table already set up
  # See /code/03-basics/aws-backend
  backend "s3" {
    bucket         = "devops-directive-tf-state"
    key            = "03-basics/web-app/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locking"
    encrypt        = true
  }
}
```
This is one of the **most important blocks** in Terraform because it tells Terraform **where to store its state**.

Let's go through it line by line.

---

# What is a backend?

When you run:

```bash
terraform apply
```

Terraform creates a file called:

```text
terraform.tfstate
```

For example:

```text
terraform/
│
├── main.tf
├── terraform.tfstate
```

This file contains information like:

```text
EC2 Instance
├── Instance ID
├── Public IP
├── Security Group
├── AMI
├── Tags
└── ...
```

Terraform uses this file to know what it created.

---

# Local backend (default)

If you don't specify a backend:

```hcl
terraform {
}
```

Terraform automatically uses the **local backend**.

Meaning:

```text
Your Laptop

main.tf
terraform.tfstate
```

Everything stays on your computer.

---

# Remote backend

Now suppose you work with three engineers.

```
Alice

Bob

You
```

If each person has their own:

```text
terraform.tfstate
```

eventually someone overwrites another person's changes.

So instead everyone stores one shared state:

```
Everyone

↓

Terraform

↓

Shared State
```

That shared state is called the **remote backend**.

---

# This block

```hcl
terraform {

  backend "s3" {

    bucket = "devops-directive-tf-state"

    key = "03-basics/web-app/terraform.tfstate"

    region = "us-east-1"

    dynamodb_table = "terraform-state-locking"

    encrypt = true

  }

}
```

is telling Terraform:

> **Don't save the state on my laptop. Save it in AWS S3 instead.**

---

# Line 1

```hcl
terraform {
```

This is Terraform's global configuration.

Inside this block you usually define:

- Terraform version
    
- Providers
    
- Backend
    

---

# backend "s3"

```hcl
backend "s3" {
```

Means

> Use an AWS S3 bucket as the backend.

Instead of:

```
Laptop

↓

terraform.tfstate
```

you now have:

```
Laptop

↓

AWS S3

↓

terraform.tfstate
```

---

# bucket

```hcl
bucket = "devops-directive-tf-state"
```

This is the S3 bucket where the state will be stored.

Think of S3 as a hard drive in AWS.

Example:

```
AWS

↓

S3

↓

devops-directive-tf-state
```

---

# key

```hcl
key = "03-basics/web-app/terraform.tfstate"
```

This is **NOT an AWS access key**.

Many beginners confuse them.

It simply means:

> Where inside the bucket should the state file live?

Suppose your bucket is

```
devops-directive-tf-state
```

Inside it Terraform creates

```
03-basics/
```

Inside

```
web-app/
```

Inside

```
terraform.tfstate
```

So the bucket looks like:

```
devops-directive-tf-state
│
└──03-basics
      │
      └──web-app
             │
             └──terraform.tfstate
```

It behaves like a folder structure, even though S3 actually stores objects rather than real directories.

---

# region

```hcl
region = "us-east-1"
```

Which AWS region contains the bucket?

For you it'll probably be

```hcl
region = "eu-west-3"
```

because that's what you've been using.

---

# dynamodb_table

```hcl
dynamodb_table = "terraform-state-locking"
```

This is one of Terraform's smartest features.

Imagine this situation.

You run

```bash
terraform apply
```

At the same time your teammate also runs

```bash
terraform apply
```

Both start changing AWS.

Both try updating the same

```
terraform.tfstate
```

Now the state can become inconsistent.

---

Terraform solves this by locking.

Before it changes anything it writes into DynamoDB.

```
Terraform

↓

DynamoDB

↓

LockID
```

If another person runs Terraform they'll get

```
State is locked
```

and must wait.

When Terraform finishes

it removes the lock.

---

# encrypt

```hcl
encrypt = true
```

This tells AWS

> Encrypt the state file before storing it.

Terraform state can contain sensitive information such as:

- resource IDs
    
- public IPs
    
- private IPs
    
- IAM role ARNs
    
- database endpoints
    

Sometimes (depending on how resources are defined) it can also contain secrets or passwords. That's why encrypting the state at rest is considered a best practice.

---

# What actually happens?

When you execute

```bash
terraform init
```

Terraform:

```
Reads

↓

backend "s3"

↓

Connects to AWS

↓

Finds bucket

↓

Downloads state

↓

Uses it
```

When you execute

```bash
terraform apply
```

Terraform:

```
Downloads state

↓

Locks DynamoDB

↓

Creates EC2

↓

Updates state

↓

Uploads state

↓

Unlocks DynamoDB
```

---

# Visualizing the whole flow

```
                You

                 │

terraform apply

                 │

                 ▼

          Terraform CLI

                 │

        Reads main.tf

                 │

                 ▼

        backend "s3"

                 │

      ┌──────────┴──────────┐

      ▼                     ▼

S3 Bucket             DynamoDB

terraform.tfstate       LockID

      │

      ▼

Calls AWS APIs

      │

      ▼

Creates EC2, ALB, RDS, etc.
```

### One subtle but important point

The backend **does not create** the S3 bucket or DynamoDB table for you.

When Terraform reaches:

```hcl
backend "s3" {
  bucket = "my-bucket"
  ...
}
```

it expects:

- the S3 bucket already exists, and
    
- the DynamoDB table already exists (if you're using state locking).
    

That's why you often see a **bootstrap** step first: use a temporary local backend to create the S3 bucket and DynamoDB table, then switch to the S3 backend and migrate the state. This is the "chicken-and-egg" problem your course mentions.

---
This is an excellent question, and it's exactly the point where many people get confused.

**The short answer is: No.** The `backend "s3"` block **does not create** the S3 bucket or DynamoDB table.

That's why your course says:

> **Assumes S3 bucket and DynamoDB table already set up**

---

# Why can't Terraform create them?

Imagine this sequence:

```hcl
terraform {
  backend "s3" {
    bucket = "my-tf-state"
  }
}
```

When you run:

```bash
terraform init
```

Terraform immediately tries to do this:

```text
1. Connect to S3
2. Find bucket "my-tf-state"
3. Download terraform.tfstate
```

But if the bucket doesn't exist yet...

```text
Terraform
    │
    ▼
S3 Bucket ❌
```

Terraform has nowhere to store or read the state.

So it cannot even start.

This is the classic **"chicken-and-egg" problem**.

---

# So how do people solve it?

They use a **bootstrap project**.

### Step 1 — Use a local backend

Don't put any backend block in your configuration yet.

Terraform uses the default local backend:

```text
Laptop
│
└── terraform.tfstate
```

---

### Step 2 — Create the S3 bucket and DynamoDB table

Your `main.tf` looks like this:

```hcl
provider "aws" {
  region = "eu-west-3"
}

resource "aws_s3_bucket" "state" {
  bucket = "abdo-terraform-state-344859353091"
}

resource "aws_dynamodb_table" "locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Run:

```bash
terraform init
terraform apply
```

Terraform creates:

```text
AWS
├── S3 Bucket
└── DynamoDB Table
```

At this point, your state is **still local**.

---

### Step 3 — Switch to the S3 backend

Now add:

```hcl
terraform {
  backend "s3" {
    bucket         = "abdo-terraform-state-344859353091"
    key            = "terraform.tfstate"
    region         = "eu-west-3"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

Run:

```bash
terraform init
```

Terraform asks:

```text
Do you want to copy the existing state to the new backend?
```

Answer:

```text
yes
```

Terraform uploads your local `terraform.tfstate` to S3.

---

# The complete flow

```text
STEP 1

Local Backend

Laptop
│
└── terraform.tfstate

        │
        ▼

Create S3 + DynamoDB

        │
        ▼

AWS

S3 Bucket
DynamoDB Table

        │
        ▼

STEP 2

Add backend "s3"

        │
        ▼

terraform init

        │
        ▼

State moves

Laptop
      │
      ▼

AWS S3
terraform.tfstate
```

---

# What about Terraform Cloud?

Terraform Cloud is different.

You **don't** create anything in AWS.

When you write:

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

HashiCorp already provides the storage.

```text
Terraform

↓

HashiCorp

↓

Encrypted State

↓

State Locking
```

No S3.

No DynamoDB.

No bootstrap.

That's why many beginners start with Terraform Cloud—it's much simpler.

---

## So for your next lab, I recommend this sequence:

1. **Terraform Cloud backend**: create an organization, a workspace, and migrate your state there. This lets you see remote state without creating any AWS storage resources.
    
2. **S3 + DynamoDB backend**: create a small bootstrap Terraform project (using the local backend) that provisions the S3 bucket and DynamoDB table. Then update your main project to use the `backend "s3"` block and migrate the state.
    

This way you'll understand **both** approaches and, more importantly, _why_ the bootstrap step is necessary for an S3 backend.





----
```json
resource "aws_instance" "instance_1" {
  ami             = "ami-011899242bb902164" # Ubuntu 20.04 LTS // us-east-1
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.instances.name]
  user_data       = <<-EOF
              #!/bin/bash
              echo "Hello, World 1" > index.html
              python3 -m http.server 8080 &
              EOF
}

resource "aws_instance" "instance_2" {
  ami             = "ami-011899242bb902164" # Ubuntu 20.04 LTS // us-east-1
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.instances.name]
  user_data       = <<-EOF
              #!/bin/bash
              echo "Hello, World 2" > index.html
              python3 -m http.server 8080 &
              EOF
}

resource "aws_security_group" "instances" {
  name = "instance-security-group"
}
```

Absolutely. This is the point where Terraform starts becoming interesting because you're describing an entire server in code. Let's go through it block by block.

---

# First, one typo

Your code has:

```hcl
resouce "aws_instance"
```

It should be:

```hcl
resource "aws_instance"
```

Another typo:

```hcl
userdata =
```

should be:

```hcl
user_data =
```

And:

```hcl
instance "aws_security_group"
```

should be:

```hcl
resource "aws_security_group"
```

---

# The first EC2

```hcl
resource "aws_instance" "instance_1" {

    ami = "ami-0e1c4170d9c01184b"

    instance_type = "t3.micro"

    security_groups = [aws_security_group.instances.name]

    user_data = <<-EOF
        #!/bin/bash
        echo "Hello, world 1" > index.html
        python3 -m http.server 8080 &
    EOF
}
```

Let's examine each line.

---

# resource

```hcl
resource
```

means:

> Terraform should **create and manage** something.

Terraform resources represent AWS objects.

Examples:

```text
resource "aws_instance"

↓

EC2
```

```text
resource "aws_s3_bucket"

↓

S3
```

```text
resource "aws_db_instance"

↓

RDS
```

---

# aws_instance

```hcl
resource "aws_instance"
```

means

Create an AWS EC2 Virtual Machine.

---

# instance_1

```hcl
"instance_1"
```

This is **Terraform's local name**.

AWS never sees this.

Terraform uses it internally.

For example

```hcl
aws_instance.instance_1.id
```

means

> Give me the ID of this EC2.

---

# ami

```hcl
ami = "ami-0e1c4170d9c01184b"
```

AMI means

**Amazon Machine Image**

Think of it like

```text
Ubuntu ISO
```

or

```text
Windows ISO
```

except it already exists inside AWS.

The AMI contains

```text
Ubuntu

Python

Kernel

Drivers

Everything needed to boot
```

When Terraform runs

```text
AMI

↓

EC2
```

AWS copies the AMI into a new virtual machine.

---

# instance_type

```hcl
instance_type = "t3.micro"
```

This determines

CPU

RAM

Network

Example

```text
t3.micro

↓

2 vCPU

1 GB RAM
```

Different instance types have different performance and pricing.

---

# security_groups

```hcl
security_groups = [
    aws_security_group.instances.name
]
```

Think of a security group as a firewall.

Without one

```text
Internet

↓

Blocked
```

With one

```text
Internet

↓

Port 8080

↓

EC2
```

Notice this part:

```hcl
aws_security_group.instances.name
```

Terraform is saying

> Attach the security group named **instances**.

This is a reference to another resource in the same configuration:

```hcl
resource "aws_security_group" "instances" {

}
```

Terraform automatically knows:

```text
Create Security Group

↓

Create EC2

↓

Attach Security Group
```

You don't have to tell Terraform the order—it figures it out from the references.

---

# user_data

This is one of the coolest features of EC2.

```hcl
user_data = <<-EOF

...

EOF
```

This is a **multi-line string**.

Terraform sends it to AWS.

AWS stores it with the instance.

When Ubuntu boots for the first time:

```text
Boot

↓

Cloud-init

↓

Execute user_data
```

It's essentially a startup script.

---

## First line

```bash
#!/bin/bash
```

This tells Linux:

Run this script with Bash.

---

## Second line

```bash
echo "Hello, world 1" > index.html
```

This creates:

```text
index.html
```

containing:

```html
Hello, world 1
```

---

## Third line

```bash
python3 -m http.server 8080 &
```

This starts Python's built-in web server.

It serves the current directory over HTTP on port `8080`.

If you later visit:

```text
http://EC2_PUBLIC_IP:8080
```

you'll see:

```text
Hello, world 1
```

The `&` at the end means:

> Run this process in the background so the startup script can finish.

---

# The second EC2

```hcl
resource "aws_instance" "instance_2"
```

is almost identical.

The only difference is:

```bash
echo "Hello, world 2"
```

Now the second server returns

```text
Hello, world 2
```

This makes it easy to see which instance handled a request later when you put a load balancer in front of them.

---

# Security Group

```hcl
resource "aws_security_group" "instances" {

    name = "instance-security-group"

}
```

This creates the firewall object.

Right now it doesn't allow anything.

Later you'll add rules such as:

```hcl
resource "aws_security_group_rule" "allow_http"
```

to permit inbound traffic on port `8080`.

---

# How Terraform sees these resources

Terraform builds a dependency graph like this:

```text
main.tf

│

├── Security Group

│

└──────► EC2 Instance 1

│

└──────► EC2 Instance 2
```

Because both EC2 instances reference:

```hcl
aws_security_group.instances.name
```

Terraform knows it must create the security group first.

---

# What happens when you run `terraform apply`

```text
Terraform

│

▼

Create Security Group

│

▼

Launch EC2 #1

│

▼

Run user_data

│

▼

Start Python Web Server

│

▼

Launch EC2 #2

│

▼

Run user_data

│

▼

Start Python Web Server
```

After both instances are running, you'll have two independent web servers. In the next part of the architecture, the **Application Load Balancer** will sit in front of them and distribute incoming requests between the two servers, which is why each one returns a different message. That difference lets you verify that load balancing is working.

----

```json
resource "aws_s3_bucket" "bucket" {
  bucket_prefix = "devops-directive-web-app-data"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "bucket_versioning" {
  bucket = aws_s3_bucket.bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "bucket_crypto_conf" {
  bucket = aws_s3_bucket.bucket.bucket
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```


# Bucket 

```hcl
resource "aws_s3_bucket" "bucket" {

    bucket_prefix="devops-directive-web-app-data"

}
```

Purpose:

Imagine your web application lets users upload

- profile pictures
    
- PDFs
    
- videos
    
- backups
    

Those files would live here.

```text
Users

↓

Upload Image

↓

S3 Bucket
```

It has nothing to do with Terraform state.

---

# Let's explain it line by line

---

## Resource

```hcl
resource "aws_s3_bucket" "bucket"
```

Create an AWS S3 bucket.

Terraform name:

```text
bucket
```

AWS name:

Generated automatically because of

```hcl
bucket_prefix
```

---

# bucket_prefix

Instead of

```hcl
bucket="mybucket"
```

they wrote

```hcl
bucket_prefix="devops-directive-web-app-data"
```

Terraform automatically creates something like

```text
devops-directive-web-app-data6fd932
```

Why?

Because S3 bucket names must be globally unique.

If they hardcoded

```text
devops-directive-web-app-data
```

thousands of students would collide with each other.

Adding a random suffix avoids that.

---

# force_destroy

```hcl
force_destroy=true
```

Normally AWS refuses to delete a bucket if it contains files.

Example

```text
Bucket

↓

photo.jpg
resume.pdf
```

Terraform destroy

↓

AWS

↓

❌

Bucket not empty

````

With

```hcl
force_destroy=true
````

Terraform first deletes every object inside the bucket.

Then deletes the bucket.

This is useful for labs.

In production you often leave this as `false` to avoid accidentally deleting important data.

---

# Versioning

```hcl
resource "aws_s3_bucket_versioning"
```

This is a separate AWS feature.

Notice

```hcl
bucket = aws_s3_bucket.bucket.id
```

This means

> Configure the bucket we created above.

Terraform automatically knows the dependency.

---

## What is Versioning?

Suppose you upload

```text
resume.pdf
```

Tomorrow you upload another

```text
resume.pdf
```

Without versioning

```text
Old file

↓

Gone forever
```

With versioning

```text
Version 1

↓

Version 2

↓

Version 3
```

AWS stores every version.

You can restore an old one.

That's why Terraform sets

```hcl
status="Enabled"
```

---

# Encryption

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration"
```

This tells AWS

> Encrypt every object automatically.

---

Notice

```hcl
bucket = aws_s3_bucket.bucket.bucket
```

This is referencing

the bucket we created earlier.

---

# Rule

```hcl
rule {

}
```

Means

Apply this rule to every uploaded object.

---

# Encryption algorithm

```hcl
sse_algorithm="AES256"
```

AES-256

is a standard symmetric encryption algorithm.

AWS encrypts files **at rest**.

Users don't notice.

Upload

↓

AWS encrypts

↓

Store

↓

Decrypt automatically when someone downloads (assuming they have permission).

---

# Dependency graph

Terraform builds

```text
aws_s3_bucket.bucket

        │

        ├──────────────► Versioning

        │

        └──────────────► Encryption
```

---
```json
data "aws_vpc" "default_vpc" {
  default = true
}

data "aws_subnet_ids" "default_subnet" {
  vpc_id = data.aws_vpc.default_vpc.id
}
```



# The simple rule

## `resource`

Means:

> **Terraform should CREATE and MANAGE this object.**

Example:

```hcl
resource "aws_instance" "web" {
    ...
}
```

Terraform does:

```text
Terraform

↓

AWS

↓

Create EC2
```

If you later run:

```bash
terraform destroy
```

Terraform deletes that EC2 because **it owns it**.

---

## `data`

Means:

> **This object already exists. Just let me read its information.**

Terraform **does not create it.**

It simply asks AWS:

> "Can you tell me about this resource?"

---

# Now let's look at your code

```hcl
data "aws_vpc" "default_vpc" {
    default = true
}
```

Notice the keyword is

```hcl
data
```

not

```hcl
resource
```

Terraform is saying:

> Find the **default VPC** in my AWS account.

AWS already created this for you when your account was created.

Terraform is **not** creating a new VPC.

---

## What is a VPC?

Think of AWS like a city.

Inside the city you have neighborhoods.

A VPC is your own private neighborhood.

```
AWS

├── VPC A
│
├── VPC B
│
└── VPC C
```

Every EC2 instance must live inside one VPC.

---

Most AWS accounts already contain one called

```
Default VPC
```

The course uses it because it's simple.

---

# What does

```hcl
default = true
```

mean?

Terraform searches AWS for

```
The VPC where

Default = True
```

It returns something like

```
vpc-0abc123def456
```

Terraform stores that information.

---

# Then

```hcl
data.aws_vpc.default_vpc.id
```

means

```
Give me the ID

of

the VPC

called

default_vpc
```

Suppose AWS returns

```
vpc-0f3289ad123
```

Then

```hcl
data.aws_vpc.default_vpc.id
```

becomes

```
vpc-0f3289ad123
```

---

# Next block

```hcl
data "aws_subnet_ids" "default_subnet" {

    vpc_id = data.aws_vpc.default_vpc.id

}
```

Again

Notice

```hcl
data
```

Terraform isn't creating subnets.

It is asking AWS

> Show me all subnets inside this VPC.

---

The flow is

```
Terraform

↓

Find Default VPC

↓

VPC ID

↓

Find all Subnets

↓

Return their IDs
```

---

Suppose AWS has

```
Default VPC

├── subnet-a
├── subnet-b
└── subnet-c
```

Terraform returns

```
[
subnet-a,
subnet-b,
subnet-c
]
```

---

Later you'll see

```hcl
subnets = data.aws_subnet_ids.default_subnet.ids
```

This tells the load balancer

```
Deploy yourself

inside

these subnets.
```

---

# Why not create a VPC ourselves?

We certainly can.

It would look like

```hcl
resource "aws_vpc" "main" {

    cidr_block = "10.0.0.0/16"

}
```

Now Terraform owns the VPC.

Running

```bash
terraform destroy
```

deletes it.

---

# Why did the instructor use `data`?

Because the goal of this lesson is

> Learn Terraform.

Not

> Learn AWS networking.

Creating a VPC correctly requires creating:

- VPC
    
- Internet Gateway
    
- Route Tables
    
- Public Subnets
    
- Private Subnets
    
- NAT Gateway
    
- Associations
    
- Routes
    

That's around **100–200 lines** of Terraform.

Instead, the instructor uses the existing default VPC so the lesson can focus on other AWS resources.

---

# Resource vs Data

Imagine AWS is a warehouse.

A **resource** is like ordering a new machine:

```
Terraform

↓

AWS

↓

Build me a new machine.
```

A **data source** is like asking the warehouse clerk:

```
Terraform

↓

AWS

↓

Where is the machine you already have?
```

Nothing new is created—you just get information about an existing object.

---

# Visual example

```
AWS Account

Default VPC
│
├── subnet-1
├── subnet-2
└── subnet-3
```

Terraform executes:

```
data "aws_vpc"

↓

Find Default VPC

↓

ID = vpc-12345

↓

data "aws_subnet_ids"

↓

Find all subnets inside vpc-12345

↓

[subnet-1, subnet-2, subnet-3]
```

Then, when you create an Application Load Balancer later, Terraform can place it into those existing subnets without having to build an entire networking stack first.

### One note for AWS provider v5+

The code from your course uses:

```hcl
data "aws_subnet_ids" "default_subnet" {
  vpc_id = data.aws_vpc.default_vpc.id
}
```

In newer versions of the AWS provider (including the version you're using), `aws_subnet_ids` has been superseded. The modern pattern is typically:

```hcl
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default_vpc.id]
  }
}
```

It serves the same purpose—retrieving the subnet IDs—but uses the current data source supported by recent provider versions.

---
```json
resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.instances.id

  from_port   = 8080
  to_port     = 8080
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```
This block defines **one firewall rule** for your Security Group.

Think of it this way:

```text
Internet
      │
      ▼
Security Group (Firewall)
      │
      ▼
EC2 Instance
```

A **Security Group** is just a container for firewall rules. By itself, it allows **no inbound traffic** (unless you add rules like this one).

---

# The whole block

```hcl
resource "aws_security_group_rule" "allow_http_inbound" {

  type              = "ingress"
  security_group_id = aws_security_group.instances.id

  from_port   = 8080
  to_port     = 8080
  protocol    = "tcp"

  cidr_blocks = ["0.0.0.0/0"]

}
```

Let's go through every line.

---

# resource

```hcl
resource "aws_security_group_rule"
```

Terraform creates a **firewall rule** inside AWS.

Notice this is **not** creating the Security Group itself.

Earlier you already had:

```hcl
resource "aws_security_group" "instances" {
    name = "instance-security-group"
}
```

That creates an **empty firewall**.

Now we're adding one rule to it.

---

# security_group_id

```hcl
security_group_id = aws_security_group.instances.id
```

This tells Terraform:

> Put this rule into the security group called `instances`.

Terraform automatically figures out the dependency:

```
Create Security Group

↓

Add Rule

↓

Launch EC2
```

---

# type

```hcl
type = "ingress"
```

This is one of the most important networking concepts.

There are only two directions.

## Ingress

Means

> Traffic **coming into** your server.

```
Internet
      │
      ▼
EC2
```

Example:

Someone opens

```
http://YOUR_PUBLIC_IP:8080
```

Their request is coming **into** your machine.

That's **ingress**.

---

## Egress

Means

> Traffic **leaving** your server.

```
EC2
      │
      ▼
Internet
```

Example:

Your EC2 runs

```bash
sudo apt update
```

Ubuntu downloads packages from the Internet.

Traffic leaves your server.

That's **egress**.

---

### Think of your house

Ingress

```
Visitor

↓

Front Door

↓

Inside House
```

Egress

```
You

↓

Front Door

↓

Outside
```

Exactly the same idea.

---

# from_port

```hcl
from_port = 8080
```

Beginning of the allowed port range.

---

# to_port

```hcl
to_port = 8080
```

End of the allowed range.

Since they're equal:

```
8080

↓

8080
```

Only one port is allowed.

---

If you wrote

```hcl
from_port = 8000
to_port   = 8100
```

Terraform would allow

```
8000

8001

8002

...

8100
```

---

# protocol

```hcl
protocol = "tcp"
```

Network communication uses protocols.

Common ones are:

|Protocol|Used for|
|---|---|
|TCP|HTTP, HTTPS, SSH|
|UDP|DNS, Voice, Games|
|ICMP|Ping|

Since your Python web server is HTTP:

```bash
python3 -m http.server 8080
```

it uses

```
TCP
```

---

# cidr_blocks

This is probably the most confusing part initially.

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

This answers:

> **Who is allowed to connect?**

---

## What is CIDR?

CIDR is just a way of describing IP address ranges.

Examples:

```
192.168.1.0/24

10.0.0.0/16

172.31.0.0/20
```

Each one represents a different network.

---

## What does

```
0.0.0.0/0
```

mean?

It means

> **Every IPv4 address on the Internet.**

Literally everyone.

```
World

↓

Allowed
```

---

Example

Someone in

Egypt

↓

Allowed

Someone in

France

↓

Allowed

Someone in

Japan

↓

Allowed

Everyone.

---

That's why web servers often have

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

Otherwise no one could visit your website.

---

## Another example

Suppose you only wanted your home network.

```
192.168.1.0/24
```

Only devices in that network could connect.

Everyone else would be blocked.

---

## Another common example

SSH

Instead of

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

you might use

```hcl
cidr_blocks = ["203.0.113.15/32"]
```

A `/32` means **one specific IP address**. Only that address could SSH into the server, which is much safer than opening SSH to the whole Internet.

---

# Why port 8080?

Remember your EC2 startup script?

```bash
python3 -m http.server 8080 &
```

That web server is listening on

```
8080
```

Without this firewall rule:

```
Internet

↓

Blocked

↓

EC2
```

Even though the Python server is running, no one can reach it.

After adding the rule:

```
Internet

↓

Port 8080 Allowed

↓

Security Group

↓

Python Web Server

↓

Hello, World
```

---

# Putting it all together

When you run `terraform apply`, Terraform creates this flow:

```
Internet
      │
      ▼
Security Group
      │
      ├── Allow TCP
      ├── Port 8080
      └── From anywhere (0.0.0.0/0)
      │
      ▼
EC2 Instance
      │
      ▼
Python HTTP Server
      │
      ▼
Hello, World
```

So when someone visits:

```
http://<EC2_PUBLIC_IP>:8080
```

their request passes through the Security Group because:

- ✅ It's **ingress** (incoming traffic).
    
- ✅ It's using **TCP**.
    
- ✅ It's on **port 8080**.
    
- ✅ Their IP matches **0.0.0.0/0** (everyone).
    

If any one of those conditions didn't match, AWS would drop the traffic before it ever reached your EC2 instance.

---
```json
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.load_balancer.arn
  port = 80
  protocol = "HTTP"
  # By default, return a simple 404 page
  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "404: page not found"
      status_code  = 404
    }
  }
}

resource "aws_lb_target_group" "instances" {
  name     = "example-target-group"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default_vpc.id
  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_target_group_attachment" "instance_1" {
  target_group_arn = aws_lb_target_group.instances.arn
  target_id        = aws_instance.instance_1.id
  port             = 8080
}

resource "aws_lb_target_group_attachment" "instance_2" {
  target_group_arn = aws_lb_target_group.instances.arn
  target_id        = aws_instance.instance_2.id
  port             = 8080
}

resource "aws_lb_listener_rule" "instances" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100
  condition {
    path_pattern {
      values = ["*"]
    }
  }
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.instances.arn
  }
}

resource "aws_security_group" "alb" {
  name = "alb-security-group"
}

resource "aws_security_group_rule" "allow_alb_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.alb.id
  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "allow_alb_all_outbound" {
  type              = "egress"
  security_group_id = aws_security_group.alb.id
  from_port   = 0
  to_port     = 0
  protocol    = "-1"
  cidr_blocks = ["0.0.0.0/0"]

}

resource "aws_lb" "load_balancer" {
  name               = "web-app-lb"
  load_balancer_type = "application"
  subnets            = data.aws_subnet_ids.default_subnet.ids
  security_groups    = [aws_security_group.alb.id]
}
```





---

# Let's first understand what we're building

Imagine users opening your website.

```text
                 Internet
                     │
                     ▼
            Application Load Balancer
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
      EC2 Instance         EC2 Instance
    Hello World 1        Hello World 2
```

The Application Load Balancer (ALB) receives every request and decides which EC2 instance should handle it.

Now let's explain each resource.

---

# 1. `aws_lb`

```hcl
resource "aws_lb" "load_balancer" {
```

This creates an **Application Load Balancer**.

Think of it as a receptionist.

Instead of users talking directly to your servers:

```text
Internet
   │
   ├──► EC2 #1
   └──► EC2 #2
```

everyone first talks to the receptionist:

```text
Internet
      │
      ▼
Load Balancer
      │
      ├──► EC2 #1
      └──► EC2 #2
```

---

## Name

```hcl
name = "web-app-lb"
```

This is simply the AWS name you'll see in the console.

---

## Type

```hcl
load_balancer_type = "application"
```

AWS supports several load balancers.

The common ones are:

```text
Application Load Balancer (HTTP/HTTPS)

Network Load Balancer (TCP/UDP)

Gateway Load Balancer
```

Since we're serving a website:

```text
HTTP

↓

Application Load Balancer
```

---

## Subnets

```hcl
subnets = data.aws_subnet_ids.default_subnet.ids
```

A load balancer cannot float in space.

It must be deployed inside one or more subnets.

Terraform asks AWS:

> Which subnets belong to my default VPC?

Then deploys the ALB there.

---

# 2. Target Group

```hcl
resource "aws_lb_target_group" "instances"
```

This is one of the most important AWS concepts.

A target group is **the list of servers behind the load balancer**.

Think of it like a contact list.

```text
Load Balancer

↓

Target Group

↓

EC2 #1

EC2 #2
```

---

## Name

```hcl
name = "example-target-group"
```

Just the AWS name.

---

## Port

```hcl
port = 8080
```

Remember your EC2 instances run:

```bash
python3 -m http.server 8080
```

The load balancer should forward requests to port **8080**.

---

## Protocol

```hcl
protocol = "HTTP"
```

Use HTTP when talking to the servers.

---

## VPC

```hcl
vpc_id = data.aws_vpc.default_vpc.id
```

The target group belongs to one VPC.

---

# Health Check

```hcl
health_check {
```

This is another core concept.

Every few seconds the load balancer asks each server:

```text
"Are you alive?"
```

Example:

```text
ALB

↓

GET /

↓

EC2
```

If the server answers

```text
200 OK
```

it's considered healthy.

---

## Path

```hcl
path = "/"
```

Visit

```text
/
```

(the home page).

---

## Matcher

```hcl
matcher = "200"
```

Expect HTTP status:

```text
200 OK
```

Anything else means unhealthy.

---

## Interval

```hcl
interval = 15
```

Every 15 seconds

check the server.

---

## Timeout

```hcl
timeout = 3
```

Wait at most

3 seconds

for the reply.

---

## Healthy Threshold

```hcl
healthy_threshold = 2
```

The server must succeed

twice

before AWS trusts it again.

---

## Unhealthy Threshold

```hcl
unhealthy_threshold = 2
```

If it fails twice

remove it from service.

---

# 3. Attachments

```hcl
resource "aws_lb_target_group_attachment"
```

This simply says:

Attach this EC2

to

this target group.

Example

```text
Target Group

↓

EC2 #1
```

Second attachment

```text
Target Group

↓

EC2 #2
```

---

Notice:

```hcl
target_id = aws_instance.instance_1.id
```

means

Attach

EC2 #1.

**You have a typo in your second attachment.**

You wrote:

```hcl
target_id = aws_instance.instance_2
```

It should be:

```hcl
target_id = aws_instance.instance_2.id
```

Terraform needs the EC2 instance ID, not the whole resource.

---

# 4. Listener

```hcl
resource "aws_lb_listener"
```

A load balancer can listen on different ports.

For example

```text
80

443

8080
```

Here

```hcl
port = 80
```

means

Listen for HTTP traffic.

---

## Default Action

```hcl
default_action
```

Suppose no routing rules match.

What should happen?

This configuration returns

```text
404
```

instead of forwarding the request.

Later you'll override this with a listener rule.

---

# 5. Listener Rule

```hcl
resource "aws_lb_listener_rule"
```

This tells the ALB

When should traffic be forwarded?

---

Condition

```hcl
values = ["*"]
```

means

Match every URL.

```text
/

/login

/about

/profile
```

Everything.

---

Action

```hcl
type = "forward"
```

means

Forward

to

the target group.

---

The flow becomes

```text
Internet

↓

ALB

↓

Listener

↓

Listener Rule

↓

Target Group

↓

EC2
```

---

# 6. Security Group for ALB

```hcl
resource "aws_security_group" "alb"
```

The load balancer itself also needs a firewall.

Remember

Everything in AWS has firewalls.

---

# Inbound Rule

```hcl
type = "ingress"
```

Allow incoming

HTTP

on

```text
80
```

from

```text
0.0.0.0/0
```

meaning

everyone.

---

# Outbound Rule

```hcl
type = "egress"
```

Allow the load balancer

to connect

to the EC2 instances.

---

## One correction

You wrote:

```hcl
protocol = "tcp"
```

for:

```hcl
from_port = 0
to_port = 0
```

The course uses:

```hcl
protocol = "-1"
```

because `-1` means **all protocols**. AWS expects that when you allow all traffic on all ports. Using `"tcp"` with ports `0-0` does **not** mean "all outbound traffic."

---

# The entire request flow

Once everything is created, a user's request travels like this:

```text
Browser

↓

http://your-domain.com

↓

Application Load Balancer
      │
      ▼
Listener (Port 80)
      │
      ▼
Listener Rule
      │
      ▼
Target Group
      │
 ┌────┴────┐
 ▼         ▼
EC2 #1   EC2 #2
```

The ALB continuously performs health checks. If EC2 #2 stops responding with `200 OK`, the flow changes automatically:

```text
Browser

↓

Load Balancer

↓

Target Group

↓

EC2 #1 ✅

EC2 #2 ❌ (removed until healthy again)
```

Users continue reaching your application without needing to know one server failed.

---

# The "secret sauce" for learning Terraform

Don't try to memorize HCL syntax. Instead, think in terms of three layers:

1. **Learn the AWS architecture first.** Ask yourself: _What AWS service do I need?_ (EC2, ALB, S3, RDS, VPC, etc.)
    
2. **Learn the relationships.** For example:
    
    - An ALB forwards to a **Target Group**.
        
    - A Target Group contains **EC2 instances**.
        
    - EC2 instances live inside a **VPC** and **Subnets**.
        
    - Security Groups protect both the ALB and the EC2 instances.
        
3. **Translate that architecture into Terraform resources.** Each AWS object usually becomes one `resource` block, and references like `aws_lb.load_balancer.arn` or `aws_instance.instance_1.id` connect them together.
    

Once you understand the architecture, the Terraform code reads more like a description of the infrastructure than a list of commands to memorize.

----
```json
resource "aws_route53_zone" "primary" {
  name = "devopsdeployed.com"
}

resource "aws_route53_record" "root" {
  zone_id = aws_route53_zone.primary.zone_id
  name    = "devopsdeployed.com"
  type    = "A"
  alias {
    name                   = aws_lb.load_balancer.dns_name
    zone_id                = aws_lb.load_balancer.zone_id
    evaluate_target_health = true
  }
}
```




This is the final piece that makes your application accessible using a **human-readable domain name** instead of the AWS-generated Load Balancer address.

Let's build the idea from scratch.

---

# Before Route 53

Suppose you create an Application Load Balancer.

AWS gives it a DNS name like:

```text
web-app-lb-123456789.eu-west-3.elb.amazonaws.com
```

Technically you can visit:

```text
http://web-app-lb-123456789.eu-west-3.elb.amazonaws.com
```

and your website works.

But that's ugly.

Instead you want:

```text
https://www.ummah.com
```

or

```text
https://ummah.com
```

That's what Route 53 is for.

---

# What is Route 53?

Route 53 is AWS's DNS (Domain Name System) service.

Think of DNS as the Internet's phone book.

Instead of remembering

```text
54.73.12.89
```

people type

```text
www.ummah.com
```

DNS translates:

```text
www.ummah.com

↓

54.73.12.89
```

or in this case:

```text
www.ummah.com

↓

Application Load Balancer
```

---

# First Resource

```hcl
resource "aws_route53_zone" "primary" {

    name = "www.ummah.com"

}
```

Actually...

**This is slightly wrong.**

A Route 53 **Hosted Zone** should usually be the root domain, not the `www` subdomain.

It should be:

```hcl
resource "aws_route53_zone" "primary" {

    name = "ummah.com"

}
```

Why?

Because the hosted zone represents the entire domain.

Inside it you create records like:

```text
ummah.com

www.ummah.com

api.ummah.com

mail.ummah.com

blog.ummah.com
```

---

## What is a Hosted Zone?

Think of it like a folder.

```text
Hosted Zone

ummah.com
```

Inside the folder:

```text
www

api

mail

shop
```

Each one is a DNS record.

---

# Second Resource

```hcl
resource "aws_route53_record" "root"
```

This creates **one DNS record**.

Think of it as one entry inside the phone book.

---

# zone_id

```hcl
zone_id = aws_route53_zone.primary.zone_id
```

Terraform says

Create this record

inside

this hosted zone.

Dependency:

```text
Hosted Zone

↓

DNS Record
```

---

# Name

```hcl
name = "www.ummah.com"
```

This is the hostname users will type.

Example

```text
www.ummah.com
```

---

# Type

```hcl
type = "A"
```

DNS has many record types.

The most common are:

|Type|Purpose|
|---|---|
|A|Maps a hostname to an IPv4 address (or an AWS alias target).|
|AAAA|Maps to an IPv6 address.|
|CNAME|Points one hostname to another hostname.|
|MX|Mail server records.|
|TXT|Verification records, SPF, DKIM, etc.|

Here you're using an **A record**.

Normally an A record looks like:

```text
www.ummah.com

↓

54.123.98.10
```

But AWS lets you use an **Alias A Record**, which points directly to certain AWS resources like an Application Load Balancer.

---

# Alias Block

Instead of storing an IP address:

```hcl
alias {
```

you're saying:

Point this record

to

another AWS resource.

---

## Name

```hcl
name = aws_lb.load_balancer.dns_name
```

Suppose AWS created:

```text
web-app-lb-123.eu-west-3.elb.amazonaws.com
```

Terraform automatically gets it.

So

```text
www.ummah.com

↓

web-app-lb-123.eu-west-3.elb.amazonaws.com
```

The user never sees the long AWS hostname.

---

## Zone ID

```hcl
zone_id = aws_lb.load_balancer.zone_id
```

Every AWS Load Balancer belongs to a Route 53 hosted zone managed by AWS.

Terraform uses this internally so Route 53 knows exactly which load balancer the alias targets.

---

## evaluate_target_health

```hcl
evaluate_target_health = true
```

This is a useful Route 53 feature.

Suppose your Application Load Balancer becomes unhealthy.

If this option is enabled, Route 53 can take the health of the target into account when answering DNS queries (particularly in routing policies involving multiple targets).

For a simple single-ALB setup, it doesn't change much, but it's good practice and becomes more valuable in more advanced DNS configurations.

---

# The complete flow

Without Route 53:

```text
Browser

↓

web-app-lb-123.eu-west-3.elb.amazonaws.com

↓

Load Balancer

↓

EC2
```

With Route 53:

```text
Browser

↓

www.ummah.com

↓

Route 53

↓

Application Load Balancer

↓

Target Group

↓

EC2
```

---

# One important thing the course doesn't emphasize

Creating:

```hcl
resource "aws_route53_zone" "primary"
```

**does not automatically make your domain work.**

If you bought `ummah.com` from another registrar (for example, a domain registrar outside AWS), Route 53 will create a hosted zone and then give you **four AWS nameservers**, something like:

```text
ns-123.awsdns-45.com
ns-456.awsdns-12.net
ns-789.awsdns-88.org
ns-101.awsdns-22.co.uk
```

You must then go to your **domain registrar** and replace the domain's nameservers with these AWS nameservers. After DNS propagation, requests for `ummah.com` and `www.ummah.com` will start being answered by Route 53.

---

# One more recommendation

For learning, I would use:

```hcl
resource "aws_route53_zone" "primary" {
    name = "ummah.com"
}

resource "aws_route53_record" "www" {
    zone_id = aws_route53_zone.primary.zone_id
    name    = "www.ummah.com"
    type    = "A"

    alias {
        name                   = aws_lb.load_balancer.dns_name
        zone_id                = aws_lb.load_balancer.zone_id
        evaluate_target_health = true
    }
}
```

This matches how DNS is normally organized:

```text
Hosted Zone
│
└── ummah.com
      │
      ├── www → Application Load Balancer
      ├── api → API server
      ├── blog → Blog server
      └── mail → Mail service
```

---

```json
resource "aws_db_instance" "db_instance" {
  allocated_storage = 20
  # This allows any minor version within the major engine_version
  # defined below, but will also result in allowing AWS to auto
  # upgrade the minor version of your DB. This may be too risky
  # in a real production environment.
  auto_minor_version_upgrade = true
  storage_type               = "standard"
  engine                     = "postgres"
  engine_version             = "12"
  instance_class             = "db.t2.micro"
  name                       = "mydb"
  username                   = "foo"
  password                   = "foobarbaz"
  skip_final_snapshot        = true
}
```




This resource creates an **Amazon RDS database**.

Before explaining the code, let's understand what RDS actually is.

---

# What is RDS?

Suppose you have a website.

```text
Users
   │
   ▼
Website
```

Where do users' data go?

Examples:

- usernames
    
- passwords
    
- posts
    
- comments
    
- products
    
- orders
    

Those need to be stored in a database.

Without RDS, you would have to install PostgreSQL yourself on an EC2 instance:

```text
EC2

Ubuntu

↓

Install PostgreSQL

↓

Configure Firewall

↓

Create Database

↓

Backups

↓

Monitoring

↓

Upgrades
```

You would manage everything yourself.

---

With Amazon RDS:

```text
AWS

↓

Creates PostgreSQL

↓

Backups

↓

Monitoring

↓

Storage

↓

Updates

↓

Availability
```

AWS manages almost everything.

That's why RDS is called a **managed database service**.

---

# Your Terraform resource

```hcl
resource "aws_db_instance" "db_instance" {
```

Terraform is saying:

> Create one RDS database instance.

Think of it like:

```text
Terraform

↓

AWS

↓

PostgreSQL Server
```

---

# allocated_storage

You wrote:

```hcl
allocate_storage = 20
```

It should be:

```hcl
allocated_storage = 20
```

This means

Give the database

```text
20 GB
```

of disk space.

---

# auto_minor_version_upgrade

You wrote:

```hcl
auto_auto_minor_version_upgrade
```

It should be:

```hcl
auto_minor_version_upgrade = true
```

This means AWS can automatically install **minor** PostgreSQL updates.

Example:

```text
PostgreSQL

12.10

↓

12.11

↓

12.12
```

Minor updates usually include:

- bug fixes
    
- security fixes
    
- performance improvements
    

It **will not** automatically upgrade from PostgreSQL 12 to PostgreSQL 13 because that's a major version upgrade.

---

# storage_type

```hcl
storage_type = "standard"
```

This chooses the type of disk.

Common options include:

```text
standard

gp2

gp3

io1

io2
```

Today, `gp3` is generally recommended for most workloads because it offers better performance and flexibility than the older `standard` or `gp2` options.

---

# engine

Your code is missing one line.

It should include:

```hcl
engine = "postgres"
```

This tells AWS what database software to install.

Options include:

```text
PostgreSQL

MySQL

MariaDB

Oracle

SQL Server
```

Here you're choosing PostgreSQL.

---

# engine_version

```hcl
engine_version = "12"
```

Install PostgreSQL version 12.

---

# instance_class

```hcl
instance_class = "db.t2.micro"
```

This is the hardware running the database.

Just like EC2 has:

```text
t3.micro

t3.small

t3.medium
```

RDS has database instance classes.

For newer accounts, you would typically use a newer generation, such as a `db.t3.micro` or `db.t4g.micro` if supported in your region, rather than the older `db.t2.micro`.

---

# name

You wrote:

```hcl
name = "mydb"
```

In newer AWS provider versions, this argument has been renamed.

You should use:

```hcl
db_name = "mydb"
```

This is the initial database that AWS creates.

Imagine PostgreSQL starts with:

```text
PostgreSQL

↓

Database

↓

mydb
```

---

# username

```hcl
username = "Abdulrahman"
```

This creates the administrator account.

Later you connect using:

```text
Username

↓

Abdulrahman
```

---

# password

```hcl
password = "testAbdo"
```

This is the administrator password.

⚠️ **In a real project, never hardcode passwords in Terraform files.**

Instead, use:

- Terraform variables marked as sensitive,
    
- environment variables,
    
- or a secrets manager such as AWS Secrets Manager.
    

The course hardcodes it only to keep the example simple.

---

# skip_final_snapshot

```hcl
skip_final_snapshot = true
```

Normally when deleting an RDS database, AWS asks:

> Should I create one final backup before deleting it?

If:

```hcl
skip_final_snapshot = false
```

AWS creates a snapshot first.

If:

```hcl
skip_final_snapshot = true
```

Terraform deletes the database immediately.

For labs, `true` is convenient.

For production databases, it's usually safer to keep a final snapshot.

---

# What happens when you run `terraform apply`?

Terraform sends a request to AWS:

```text
Create a PostgreSQL database

20 GB Storage

↓

Engine = PostgreSQL

↓

Version = 12

↓

Instance Size = db.t2.micro

↓

Database = mydb

↓

Admin User = Abdulrahman
```

After several minutes, AWS returns something like:

```text
mydb.xxxxxx.eu-west-3.rds.amazonaws.com
```

That's your database endpoint.

Your application can then connect to it using:

```text
Host:
mydb.xxxxxx.eu-west-3.rds.amazonaws.com

Port:
5432

Username:
Abdulrahman

Password:
********
```

---

# The overall architecture

After adding RDS, your infrastructure looks like this:

```text
                  Internet
                      │
                      ▼
          Application Load Balancer
                      │
             ┌────────┴────────┐
             ▼                 ▼
        EC2 Instance      EC2 Instance
             │                 │
             └────────┬────────┘
                      │
                      ▼
             Amazon RDS (PostgreSQL)
```

The EC2 instances handle incoming web requests, while the RDS instance stores the application's persistent data.

