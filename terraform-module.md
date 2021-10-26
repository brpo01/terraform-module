# AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM. PART 3 – REFACTORING

In this documentation, we'll be refactoring our terraform code from the last [project](https://github.com/brpo01/terraform-2/blob/master/terraform.md) into modules to make them reusable. We'll also be configuring aws s3 bucket as a remote backend to store state files and DynamoDB for state locking & consistency checking.

## Introducing Backend on AWS S3

Each Terraform configuration can specify a backend, which defines where and how operations are performed, where state snapshots are stored, etc.
Take a peek into what the states file looks like. It is basically where terraform stores all the state of the infrastructure in json format.

So far, we have been using the default backend, which is the local backend – it requires no configuration, and the states file is stored locally. This mode can be suitable for learning purposes, but it is not a robust solution, so it is better to store it in some more reliable and durable storage.

The second problem with storing this file locally is that, in a team of multiple DevOps engineers, other engineers will not have access to a state file stored locally on your computer.

To solve this, we will need to configure a backend where the state file can be accessed remotely other DevOps team members. There are plenty of different standard backends supported by Terraform that you can choose from. Since we are already using AWS – we can choose an S3 bucket as a backend.

Another useful option that is supported by S3 backend is State Locking – it is used to lock your state for all operations that could write state. This prevents others from acquiring the lock and potentially corrupting your state. State Locking feature for S3 backend is optional and requires another AWS service – DynamoDB.

Here is our plan to Re-initialize Terraform to use S3 backend:

- Add S3 and DynamoDB resource blocks before deleting the local state file.
- Update terraform block to introduce backend and locking
- Re-initialize terraform
- Delete the local tfstate file and check the one in S3 bucket
- Add outputs
- Run "terraform apply"

Now let us begin configuring the remote backend

- Create a file and name it backends.tf. Add the below code. Before you initialize create an s3 bucket resource on your aws account and use the name you specified in your code - "dev-terraform-bucket". You must also be aware that Terraform stores secret data inside the state files. Passwords, and secret keys processed by resources are always stored in there. Hence, you must consider to always enable encryption. You can see how we achieved that with server_side_encryption_configuration.

```
# Note: The bucket name may not work for you since buckets are unique globally in AWS, so you must give it a unique name.
resource "aws_s3_bucket" "terraform_state" {
  bucket = "dev-terraform-bucket"
  # Enable versioning so we can see the full revision history of our state files
  versioning {
    enabled = true
  }
  # Enable server-side encryption by default
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}
```
- Create a DynamoDB table to handle locks and perform consistency checks. In previous projects, locks were handled with a local file as shown in terraform.tfstate.lock.info. Since we now have a team mindset, causing us to configure S3 as our backend to store state file, we will do the same to handle locking. Therefore, with a cloud storage database like DynamoDB, anyone running Terraform against the same infrastructure can use a central location to control a situation where Terraform is running at the same time from multiple different people.

```
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```
- Run **terraform apply** to create both resources. Terraform expects that both S3 bucket and DynamoDB resources are already created before we configure the backend. So, let us run terraform apply to provision resources.

- Configure S3 Backend

```
terraform {
  backend "s3" {
    bucket         = "dev-terraform-bucket"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

- Now its time to re-initialize the backend. Run terraform init and confirm you are happy to change the backend by typing yes. You have successfully migrated your state file form your local machine to a remote s3 bucket. Terraform will automatically read the latest state from the S3 bucket to determine the current state of the infrastructure.

- Add the code below to the outputs.tf file

```
output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
  description = "The ARN of the S3 bucket"
}
output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_locks.name
  description = "The name of the DynamoDB table"
}
```

## Terraform Modules and best practices to structure your .tf codes

Modules serve as containers that allow to logically group Terraform codes for similar resources in the same domain (e.g., Compute, Networking, AMI, etc.). One root module can call other child modules and insert their configurations when applying Terraform config. This concept makes your code structure neater, and it allows different team members to work on different parts of configuration at the same time.

You can also create and publish your modules to Terraform Registry for others to use and use someone’s modules in your projects.

Module is just a collection of .tf and/or .tf.json files in a directory.

You can refer to existing child modules from your root module by specifying them as a source, like this:

```
module "network" {
  source = "./modules/network"
}
```

## Refactor Your Project Using Modules

- Break down your Terraform codes to have all resources in their respective modules. Combine resources of a similar type into directories within a ‘modules’ directory, for example, like this:

```
  - Loadbalancing
  - EFS
  - RDS
  - Autoscaling
  - Compute
  - Networking
  - Security
  - Certificate
```
 - Each module shall contain following files:

```
- main.tf (or %resource_name%.tf) file(s) with resources blocks
- outputs.tf (optional, if you need to refer outputs from any of these resources in your root module)
- variables.tf (it is a good practice not to hard code the values and use variables)
```

- It is also recommended to configure providers and backends sections in separate files. Now let us break our terraform code into modules

### Compute Module

- Create a folder called compute and add these three files - main.tf, variables.tf & outputs.tf

- Move roles.tf & the launch templates into the main.tf file in the compute folder.

**main.tf**
```
resource "aws_iam_role" "ec2_instance_role" {
  name = "ec2_instance_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = merge(
    var.tags,
    {
      Name = "aws assume role"
    },
  )
}

resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]

  })

  tags = merge(
    var.tags,
    {
      Name =  "aws assume policy"
    },
  )

}

resource "aws_iam_role_policy_attachment" "test-attach" {
    role       = aws_iam_role.ec2_instance_role.name
    policy_arn = aws_iam_policy.policy.arn
}

resource "aws_iam_instance_profile" "ip" {
    name = "aws_instance_profile_test"
    role =  aws_iam_role.ec2_instance_role.name
}

data "aws_availability_zones" "available" {}

resource "random_shuffle" "az_list" {
  input    = data.aws_availability_zones.available.names
}

# ---- Launch templates for bastion  hosts

resource "aws_launch_template" "bastion-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.bastion-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
    resource_type = "instance"

   tags = merge(
    var.tags,
    {
      Name = "bastion-launch-template"
    },
  )
  }

  user_data = var.bastion_user_data
}

#--------- launch template for nginx

resource "aws_launch_template" "nginx-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.nginx-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name =  var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
    resource_type = "instance"

    tags = merge(
    var.tags,
    {
      Name = "nginx-launch-template"
    },
  )
  }

  user_data = var.nginx_user_data
}

# launch template for wordpress

resource "aws_launch_template" "wordpress-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.webserver-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
    resource_type = "instance"

    tags = merge(
    var.tags,
    {
      Name = "wordpress-launch-template"
    },
  )

  }

  user_data = var.wordpress_user_data
}

# launch template for toooling
resource "aws_launch_template" "tooling-launch-template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [var.webserver-sg]

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }

  key_name = var.keypair

  placement {
    availability_zone = "random_shuffle.az_list.result"
  }

  lifecycle {
    create_before_destroy = true
  }

  tag_specifications {
    resource_type = "instance"

  tags = merge(
    var.tags,
    {
      Name = "tooling-launch-template"
    },
  )

  }

  user_data = var.tooling_user_data
}
```

**variables.tf**

```
variable "ami" {}

variable "bastion-sg" {}

variable "nginx-sg" {}

variable "webserver-sg" {}

variable "bastion_user_data" {}

variable "nginx_user_data" {}

variable "tooling_user_data" {}

variable "wordpress_user_data" {}

variable "keypair" {}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```

- Add outputs in the outputs.tf. We'll be referencing these outputs in the root main.tf file. Also create a user-data folder for your user-data scripts 

**outputs.tf**

```
output "bastion_launch_template" {
    value = aws_launch_template.bastion-launch-template.id
}

output "nginx_launch_template" {
    value = aws_launch_template.nginx-launch-template.id
}

output "wordpress_launch_template" {
    value = aws_launch_template.wordpress-launch-template.id
}

output "tooling_launch_template" {
    value = aws_launch_template.tooling-launch-template.id
}
```

**root main.tf**

```
module "compute" {
  source = "./compute"
  ami = var.ami
  bastion-sg = module.security.bastion
  nginx-sg = module.security.nginx
  webserver-sg = module.security.webservers
  keypair = var.keypair
  bastion_user_data = filebase64("${path.module}/user-data/bastion.sh")
  nginx_user_data = filebase64("${path.module}/user-data/nginx.sh")
  wordpress_user_data = filebase64("${path.module}/user-data/wordpress.sh")
  tooling_user_data = filebase64("${path.module}/user-data/tooling.sh")
}
```

### Networking Module

- Create a folder called networking and add these three files - main.tf, variables.tf & outputs.tf.

- Move internetgateway.tf, natgateway.tf into the main.tf file in the compute folder. Also move the the VPC & subnets originally in the root folder into the networking folder.

**main.tf**

```
# Get list of availability zones
data "aws_availability_zones" "available" {
    state = "available"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support
  enable_dns_hostnames           = var.enable_dns_hostnames
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink_dns_support

  tags = merge(
    var.tags,
    {
      Name = "main-vpc"
    }
  )

  lifecycle {
    create_before_destroy = true 
  }
}

# Create public subnets
resource "aws_subnet" "public_subnet" {
  count = var.public_sn_count
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_cidr[count.index]
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

  tags = merge(
    var.tags,
    {
      Name = format("public-subnet-%s", count.index)
    }
  )
}

# Create public subnets
resource "aws_subnet" "private_subnet" {
  count = var.private_sn_count
  vpc_id = aws_vpc.main.id
  cidr_block = var.private_cidr[count.index]
  map_public_ip_on_launch = true
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(
    var.tags,
    {
      Name = format("private-subnet-%s", count.index)
    }
  )
}

# internet gateway
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-%s!", "ig-",aws_vpc.main.id)
    } 
  )
}

# Elastic ip resource
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.main_igw]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP-%s", var.name, var.environment)
    },
  )
}

# nat gateway resource
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public_subnet.*.id, 0)
  depends_on    = [aws_internet_gateway.main_igw]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat-%s", var.name, var.environment)
    },
  )
}

# create private route table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# associate all private subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private_subnet[*].id)
  subnet_id      = element(aws_subnet.private_subnet[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}

# create route for the private route table and attach the nat gateway
resource "aws_route" "private-rtb-route" {
  route_table_id         = aws_route_table.private-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_nat_gateway.nat.id
}

# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}

# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public_subnet[*].id)
  subnet_id      = element(aws_subnet.public_subnet[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}

# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main_igw.id
}
```

**variables.tf**

```
variable "vpc_cidr" {
  type = string
  description = "The VPC cidr"
}

variable "enable_dns_support" {
  type = bool
}

variable "enable_dns_hostnames" {
  type = bool
}

variable "enable_classiclink" {
  type = bool
}

variable "enable_classiclink_dns_support" {
  type = bool
}

variable "public_sn_count" {
  type        = number
  description = "Number of public subnets"
}

variable "private_sn_count" {
  type        = number
  description = "Number of private subnets"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}

variable "public_cidr" {}

variable "private_cidr" {}

variable "name" {
  type    = string
  default = "main"
}

variable "environment" {
  type = string
  default = "production"
}
```

- Add outputs in the outputs.tf. We'll be referencing these outputs in the root main.tf file. 

```
output "vpc_id" {
    value = aws_vpc.main.id
}

output "public_subnet0" {
    value = aws_subnet.public_subnet[0].id
}

output "public_subnet1" {
    value = aws_subnet.public_subnet[1].id
}

output "private_subnet0" {
    value = aws_subnet.private_subnet[0].id
}

output "private_subnet1" {
    value = aws_subnet.private_subnet[1].id
}

output "private_subnet2" {
    value = aws_subnet.private_subnet[2].id
}

output "private_subnet3" {
    value = aws_subnet.private_subnet[3].id
}
```

**root main.tf**

```
```