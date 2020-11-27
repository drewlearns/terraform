# TERRAFORM NOTES
_I took a course on Terraform from udemy ([Link to course](https://www.udemy.com/share/101X20AEMdeF9QQX0B/)).  As usual, I wrote some notes for it and thought I'll need them later and share them for posterity. Enjoy 🙃_ 
 
-- [DrewLearns](https://twitter.com/drewlearns2)

---

## Useful Commands You'll need to be familiar with
* $ `terraform init` : 
    This installs the hashicorp plugins in the appropriate directories. If you ever set or change modules or backend configuration for Terraform, rerun this command to reinitialize your working directory. If you forget, other commands will detect it and remind you to do so if necessary.
* $ `terraform plan`:
    use this command to see any changes that are required for your infrastructure. This is like a dry run.
    >_Note_: You didn't specify an "-out" parameter to save this plan, so Terraform can't guarantee that exactly these actions will be performed if "terraform apply" is subsequently run.

* $ `terraform apply`: apply is a shortcut for plan & apply - avoid this in production. Command is used to "apply" the terraform configuration to the service provider.

* $ `terraform plan -out out.terraform`: 
    plan out terraform plan runs and then writes the plan to out file "out.terraform" (or what ever you want to call it).

* $ `terraform apply out.terraform`: 
    "apply out" terraform plan using that out.terraform file you created above

* $ `terraform show`: 
    shows current state

* $ `cat terraform.tfstate`: 
    tfstate show state in JSON format

* $ `terraform destroy` : Usage: terraform destroy [options] [DIR] "Destroy Terraform-managed infrastructure"

```
$ cat terraform.tfstate
{
  "version": 4,
  "terraform_version": "0.13.5",
  "serial": 1,
  "lineage": "caffa773-42aa-f5a6-b0df-c1771ac1ac7e",
  "outputs": {},
  "resources": []
}
```
---

## Setup your local environment:
>_Note_: If you don't already have brew installed, I'd recommend it. These instructions are for a mac computer, linux maybe. Windows users, these instructions will differ drastically.

$ `brew tap hashicorp/tap &&  brew install hashicorp/tap/terraform`
* Installs terraform

$ `brew upgrade hashicorp/tap/terraform`
* ensure’s it’s the most up to date version

$ `terraform -help`
* use this to verify terraform is installed

Log into AWS, if you haven’t already and create a new user/iam role in AWS console. Now log into AWS in your terminal
$ `aws configure`
* logs you into aws

---

# Setup some files
## Create a provider.tf file
* This file is used to manage who the provider is and what AZ it will create resources on.
* This is a file where all provider variables will be kept (secrets and the region). Use “${var.AWS_ACCESS_KEY}” for example in place of the actual key.
* It should look like this:

```
provider “aws” {
	access_key	= “${var.AWS_ACCESS_KEY}”
	secret_key	= “${var.AWS_SECRET_KEY}”
	region		= “us-east-1
}
```

---

## Create a vars.tf file
* Use this file to declare the variables described above in the provider.tf file.
* It should look like this:

```
variable "AWS_ACCESS_KEY" {}
variable "AWS_SECRET_KEY" {} variable "AWS_REGION" {
    default = "us-east-1"   
}
```

* Since the access and secret curly brackets are empty, we are not defining a value and/or if you left those values empty in terraform.tfvars (didn’t call them), you’d be prompted for the access and secret when running terraform apply/plan

---

## Create a .gitignore file
In the gitignore, you want to add terraform.tfvars - this is the file where we will store our secrets.

---

## Create a terraform.tfvars file 
* It should look like this:

```
AWS_ACCESS_KEY = ""
AWS_SECRET_KEY = ""
AWS_REGION = ""
```
---

## Create an instance.tf file
* Note: “lookup” will search for a value in a map. Bear in mind, this is looking up a value in a map that doesn’t exist yet, we’ll fix that in a moment.

```
resource “aws_instance” “example” {
	ami			= “${lookup(var.AWS_AMIS, var.AWS_REGION)}”
	instance_type  = “t2.micro”
}
```

Update the vars.tf file to include the map above
* Use https://cloud-images.ubuntu.com/locator/ec2/ to find an AMI

```
variable "AWS_ACCESS_KEY" {}
variable "AWS_SECRET_KEY" {} variable "AWS_REGION" {
    default = "us-east-1"   
}
variable "AWS_AMIS" {
    type = map
    default = {
        us-east-1 = "ami-022758574f5a26580"
        us-east-2 = "ami-0b8bee0a15442a995"
    }
}
```

---

## TERRAFORM IT
Initialize terraform: $ `terraform init`
Plan terraform:
$ `terraform plan`
If you are satisfied with the plan, you can “plan out” (save the plan to ensure it runs as planned).
$ `terraform plan out filename`
Apply the terraform plan:
$ `terraform apply -out filename`

---

# PROVISIONING SOFTWARE
## Terraform isn't software managment
2 ways to provision software on the hardware you have provisioned:
1. Create your own AMI and bundle our own software with the image
    1. Packer is a great tool for this
2. Boot standardized AMIs and then install software on it as needed. 
    * You can use automation tools like ansible, chef, or puppet or with file uploads
    * Terraform supports chef natively
    * You can upload using scripts and ssh in to use remote exec

## File uploads:
In the instance.tf file you can update it to include:

```
resource “aws_instance” “example” {
	ami			= “${lookup(var.AWS_AMIS, var.AWS_REGION)}”
	instance_type  = “t2.micro”
    provisioner "file" {
        source      = "app.conf"
        destination = "etc/myapp.conf"
    }
}
```

On Linux machines, SSH may be required to upload so you will need to specify the connection like so:

```
resource “aws_instance” “example” {
	ami			= “${lookup(var.AWS_AMIS, var.AWS_REGION)}”
	instance_type  = “t2.micro”
    provisioner "file" {
        source      = "app.conf"
        destination = "etc/myapp.conf"
        connection  = {
            user = "${var.instance_username}"
            password = "${var.instance_password}"
        }
    }
}
```

>_NOTE_: When spinning up new linux, ubuntu, or amazon AMI on AWS, the default username for SSH is “ec2-user”
Typically, you’ll use SSH key pairs, this is an alternative to passwords

```
resource "aws_key_pair" "drew-key"{
    key_name = "mykey"
    public_key = "ssh-rsa my-public-key"
}resource “aws_instance” “example” {
	ami			= “${lookup(var.AWS_AMIS, var.AWS_REGION)}”
	instance_type  = “t2.micro”
    key_name = "${aws_keypair.mykey.key_name}"
    provisioner "file" {
        source      = "script.sh"
        destination = "etc/myapp.conf"
        connection  = {
            user = "${var.instance_username}"
            password = "${file(${var.path_to_private_key})}"
        }
    }
}
```

After you have uploaded a script like above, you can remotely execute it.

```
resource "aws_key_pair" "drew-key"{
    key_name = "mykey"
    public_key = "ssh-rsa my-public-key"
}resource “aws_instance” “example” {
	ami			= “${lookup(var.AWS_AMIS, var.AWS_REGION)}”
	instance_type  = “t2.micro”
    key_name = "${aws_keypair.mykey.key_name}"
    provisioner "file" {
        source      = "script.sh"
        destination = "opt/script.sh"
        connection  = {
            user = "${var.instance_username}"
            password = "${file(${var.path_to_private_key})}"
        }
    }
    provisioner "remote-exec"{
        inline = [ 
            "chmod + x /opt/script.sh",
            "/opt/script.sh arguments"
        ]
    }
}
```

## SSH KEYGEN
To assign a key initially, you'll want to create a key using `ssh-keygen -f mykey` this will create a private and public key. Update the vars.tf files to match the .pub file. 

```
variable "PATH_TO_PUBLIC_KEY" {
  default = "mykey.pub" 
}
```

>Alternatively, you can create your IAM role to utilize your `id_rsa.pub` key pair and update the `mykey.pub` to read `~/.ssh/id_rsa.pub`


---

# TROUBLESHOOTING TIPS:
## 403 errors:
- 403 errors can sometimes be caused because the date/time is incorrect. You can use `sudo apt-get install ntpdate ; sudo ntpdate ntp ubuntu.com` 
