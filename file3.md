How to go through Terrraform:(lab1)
Absolutely. Since you're on **Arch Linux**, I'll guide you through this as if we were sitting together at the terminal. I'll also point out what is **good practice** versus what the course is doing.

---

# Step 1 — Install Terraform on Arch

There are two ways.

### Option 1 (Recommended)

```bash
sudo pacman -S terraform
```

Check that it installed:

```bash
terraform version
```

Example output

```text
Terraform v1.13.x
```

---

If your repositories don't contain Terraform, install it from the AUR.

Using yay:

```bash
yay -S terraform
```

---

# Step 2 — Install AWS CLI

Install it

```bash
sudo pacman -S aws-cli
```

Verify

```bash
aws --version
```

Example

```text
aws-cli/2.31.x
```

---

# Step 3 — Create an IAM User

Now we need AWS credentials.

Do **not** use your root AWS account.

Instead create a new IAM user.

Open AWS Console

Go to

```
IAM
```

On the left

```
Users
```

Click

```
Create user
```

Example

```
terraform-user
```

Click Next.

---

## Attach permissions

Your course suggests giving:

- AmazonEC2FullAccess
    
- AmazonS3FullAccess
    
- AmazonRDSFullAccess
    
- AmazonDynamoDBFullAccess
    
- IAMFullAccess
    
- Route53FullAccess
    

For this lab, you only need **EC2**.

So choose

```
Attach policies directly
```

Search

```
AmazonEC2FullAccess
```

Check it.

Click

```
Next
```

then

```
Create user
```

---

# Step 4 — Create Access Keys

Open your new user.

Go to

```
Security credentials
```

Scroll to

```
Access keys
```

Click

```
Create access key
```

Choose

```
Command Line Interface (CLI)
```

Continue.

AWS will show

```
Access Key ID

Secret Access Key
```

Download them or copy them.

	The secret key will never be shown again.(yes copy them)

---

# Step 5 — Configure AWS CLI

Run

```bash
aws configure
```

It asks

```text
AWS Access Key ID:
```

Paste yours.

Then

```text
AWS Secret Access Key:
```

Paste yours.

Then

```text
Default region name:
```

Example

```
us-east-1
```

Then

```text
Default output format:
```

Just type

```
json
```

---

It creates

```text
~/.aws/
```

Inside

```text
~/.aws/credentials
```

Example

```
[default]
aws_access_key_id=AKIA....
aws_secret_access_key=xxxxxxxx
```

and

```
~/.aws/config
```

Example

```
[default]
region=us-east-1
output=json
```

---

# Step 6 — Verify AWS CLI

Run

```bash
aws sts get-caller-identity
```

You should get something like

```json
{
  "UserId": "...",
  "Account": "...",
  "Arn": "arn:aws:iam::123456789:user/terraform-user"
}
```

If you see that,

✅ AWS CLI works.

Terraform will automatically use these credentials.


When you run:

```bash
sudo aws configure
```

the credentials are stored in:

```text
/root/.aws/credentials
```

Terraform, when run as your normal user (`abdo`), looks for:

```text
/home/User/.aws/credentials
```

and doesn't find it.

## Step 1: Confirm where the credentials are

Run:

```bash
sudo ls -l /root/.aws
```

and

```bash
sudo cat /root/.aws/credentials
```

You should see your access keys.

---

## Step 2 (Recommended): Copy them to your user's home

Create your own `.aws` directory:

```bash
mkdir -p ~/.aws
```

Copy the files:

```bash
sudo cp /root/.aws/credentials ~/.aws/
sudo cp /root/.aws/config ~/.aws/
```

Now make yourself the owner:

```bash
sudo chown -R $USER:$USER ~/.aws
```

Restrict the permissions (AWS recommends this):

```bash
chmod 700 ~/.aws
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config
```

Verify:

```bash
cat ~/.aws/credentials
```

You should now see:

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ...
```

Then test:

```bash
aws sts get-caller-identity
```

If that works **without `sudo`**, Terraform will be able to authenticate.

---

## Even better: Delete the root credentials

After you've confirmed everything works, you can remove the root copy so you don't accidentally use it in the future:

```bash
sudo rm -rf /root/.aws
```


---

# Step 7 — Create your Terraform project

Make a directory

```bash
mkdir terraform-lab
```

Go inside

```bash
cd terraform-lab
```

Create the file

```bash
touch main.tf
```

Open it

```bash
nano main.tf
```

Paste

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-011899242bb902164"
  instance_type = "t2.micro"
}
```

Save.

---

# Step 8 — Initialize Terraform

Run

```bash
terraform init
```

Terraform downloads the AWS provider.

You should see

```
Initializing provider plugins...

Terraform has been successfully initialized!
```

It creates

```
.terraform/
```

and

```
terraform.lock.hcl
```

---

# Step 9 — Validate

Run

```bash
terraform validate
```

Expected

```
Success!
```

---

# Step 10 — See the execution plan

Run

```bash
terraform plan
```

Example

```
Terraform will perform the following actions:

+ aws_instance.example
```

Nothing has been created yet.

Terraform is just showing you what it intends to do.
When Terraform says:

```
Reusing previous version of hashicorp/aws from the dependency lock fileUsing previously-installed hashicorp/aws v5.100.0
```

it means:

- Your configuration allows any provider version compatible with `~> 5.92`.
- `5.100.0` satisfies that constraint.
- Terraform already downloaded `5.100.0`, so it reused it instead of downloading another copy.
---

# Step 11 — Create the EC2 instance

Run

```bash
terraform apply
```

It asks

```
Do you want to perform these actions?

yes
```

Type

```
yes
```

Terraform now calls the AWS API.

After a minute you'll see

```
Apply complete!

Resources: 1 added
```

---

# Step 12 — Verify

Open the AWS Console.

Go to

```
EC2
```

Then

```
Instances
```

You'll see your instance running.

---

# Step 13 — Inspect Terraform state

Run

```bash
terraform state list
```

Output

```
aws_instance.example
```

Show details

```bash
terraform show
```

or

```bash
terraform state show aws_instance.example
```

You'll see the instance ID, public IP (if assigned), subnet, security groups, and other attributes.

---

# Step 14 — Destroy the infrastructure

To avoid charges, run

```bash
terraform destroy
```

Confirm

```
yes
```

Terraform deletes the EC2 instance.

---

# useful commands



```bash
terraform init
terraform validate
terraform fmt
terraform plan
terraform apply
terraform show
terraform state list
terraform destroy
```

---

# Understanding what happened

Terraform follows a simple workflow:

```
main.tf
     │
     ▼
terraform init
     │
Downloads AWS provider
     │
     ▼
terraform plan
     │
Calculates changes
     │
     ▼
terraform apply
     │
Calls AWS APIs
     │
     ▼
EC2 instance created
     │
Terraform records the resource in terraform.tfstate
     │
     ▼
terraform destroy
     │
Calls AWS APIs again
     │
Deletes the instance
```


  
