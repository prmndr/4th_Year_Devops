# Practical Lab 5: Setting up a Web Application Infrastructure on AWS with Terraform

## Objective
In this lab, you will use Terraform to set up a basic web application infrastructure on AWS. This will include a VPC, public and private subnets, an EC2 instance, and an RDS database.

## Prerequisites
- AWS account with appropriate permissions
- AWS CLI installed and configured with your credentials
- Terraform installed on your local machine
- Basic understanding of AWS services and networking concepts

## Estimated Time
1 hour

## Use Case
You're a DevOps engineer tasked with creating a scalable and secure infrastructure for a web application. The application needs a web server accessible from the internet and a database server in a private subnet.

## Steps

### 1. Set up the project structure

1. Create a new directory for your project:
   ```
   mkdir terraform-aws-webapp
   cd terraform-aws-webapp
   ```

2. Create the following file structure:
   ```
   terraform-aws-webapp/
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   └── userdata.sh
   ```

### 2. Create the Terraform configuration files

1. Create `variables.tf`:

```hcl
variable "aws_region" {
  description = "The AWS region to create resources in"
  default     = "ap-south-1"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  description = "CIDR block for public subnet"
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for private subnet"
  default     = "10.0.2.0/24"
}

variable "db_username" {
  description = "Database administrator username"
  type        = string
  sensitive   = true
}

variable "db_password" {
  description = "Database administrator password"
  type        = string
  sensitive   = true
}
```

2. Create `main.tf`:

```hcl
provider "aws" {
  region = "ap-south-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true

  tags = {
    Name = "Main VPC"
  }
}

# Create Public Subnet
resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidr
  availability_zone = "${var.aws_region}a"  # e.g., ap-south-1a

  tags = {
    Name = "Public Subnet"
  }
}

# Create Private Subnet
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = "${var.aws_region}b"  # e.g., ap-south-1b

  tags = {
    Name = "Private Subnet"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "Main IGW"
  }
}

# Create Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "Public Route Table"
  }
}

# Associate Public Subnet with Public Route Table
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Create Security Group for EC2
resource "aws_security_group" "web" {
  name        = "Allow HTTP"
  description = "Allow HTTP inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from VPC"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Allow HTTP"
  }
}

# Create EC2 Instance
resource "aws_instance" "web" {
  ami           = "ami-08ee1453725d19cdb"  # Amazon Linux 2 AMI (HVM), SSD Volume Type
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id

  vpc_security_group_ids = [aws_security_group.web.id]

  associate_public_ip_address = true

  user_data = file("userdata.sh")

  tags = {
    Name = "Web Server"
  }
}

# Create Security Group for RDS
resource "aws_security_group" "db" {
  name        = "Allow MySQL"
  description = "Allow MySQL inbound traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
  description = "SSH from your IP"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["3.110.31.29/32"]  # Replace with your IP
}

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Allow MySQL"
  }
}

# Create RDS Instance
resource "aws_db_instance" "default" {
  allocated_storage    = 10
  engine               = "mysql"
  engine_version       = "5.7"
  instance_class       = "db.t3.micro"
  db_name              = "mydb"
  username             = "admin"
  password             = "admin541" # minimum 8 characters
  parameter_group_name = "default.mysql5.7"
  skip_final_snapshot  = true

  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.default.name
}

# Create DB Subnet Group
resource "aws_db_subnet_group" "default" {
  name       = "main"
  subnet_ids = [aws_subnet.private.id, aws_subnet.public.id]

  tags = {
    Name = "My DB subnet group"
  }
}

```

3. Create `outputs.tf`:

```hcl
output "web_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "db_endpoint" {
  description = "Endpoint of the RDS instance"
  value       = aws_db_instance.default.endpoint
}
```

4. Create `userdata.sh`:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello World from AWS EC2 instance!</h1>" > /var/www/html/index.html
```

### 3. Initialize and apply the Terraform configuration

1. Initialize Terraform:
   ```
   terraform init
   ```

2. Set the database username and password as environment variables:
   ```
   $env: TF_VAR_db_username=admin
   $env: TF_VAR_db_password=your_secure_password
   ```

3. Review the planned changes:
   ```
   terraform plan
   ```

4. Apply the configuration:
   ```
   terraform apply
   ```

   When prompted, type `yes` to confirm the changes.

### 4. Test the web server

1. Once Terraform has finished applying the changes, note the `web_public_ip` output.

2. Use a web browser to access the EC2 instance:
   ```
   http://<web_public_ip>
   ```

   You should see the "Hello World" message.

### 5. Clean up

When you're done experimenting, destroy the created resources:

```
terraform destroy
```

When prompted, type `yes` to confirm.

## Conclusion
In this lab, you've learned how to use Terraform to set up a basic web application infrastructure on AWS, including a VPC with public and private subnets, an EC2 instance serving as a web server, and an RDS instance for the database.

## Additional Exercises
1. Add an Elastic Load Balancer in front of the EC2 instance.
2. Set up an Auto Scaling Group for the EC2 instances.
3. Use Terraform modules to organize your code better.
4. Add CloudWatch alarms for monitoring the EC2 and RDS instances.

Remember to explore the AWS and Terraform documentation for more advanced features and best practices in infrastructure management.
