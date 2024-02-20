# terraform
my Terraform code

provider "aws" {
  region = "us-east-1"
}
resource "aws_vpc" "wordpress_vpc" {
  cidr_block = var.vpc_cidr_block
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "wordpress-vpc"
  }
}
resource "aws_internet_gateway" "wordpress_igw" {
  vpc_id = aws_vpc.wordpress_vpc.id
  tags = {
    Name = "wordpress_igw"
  }
}
resource "aws_route_table" "wordpress_rt" {
  vpc_id = aws_vpc.wordpress_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.wordpress_igw.id
  }
  tags = {
    Name = "wordpess-rt"
  }
}
resource "aws_route_table_association" "public_subnet" {
  for_each = aws_subnet.public_subnet
  subnet_id      = each.value.id
  route_table_id = aws_route_table.wordpress_rt.id
}
resource "aws_subnet" "public_subnet" {
  for_each = { for idx, az in var.availability_zones : idx => az }
  cidr_block = "10.0.${each.key + 1}.0/24"
  vpc_id = aws_vpc.wordpress_vpc.id
  map_public_ip_on_launch = true
  availability_zone = each.value
  tags = {
    Name = "public-subnet-${each.key + 1}"
  }
}
resource "aws_subnet" "private_subnet" {
  for_each = { for idx, az in var.availability_zones : idx => az }
  cidr_block = "10.0.${each.key + 4}.0/24"
  vpc_id = aws_vpc.wordpress_vpc.id
  availability_zone = each.value
  tags = {
    Name = "private-subnet-${each.key + 1}"
  }
}
resource "aws_security_group" "wordpress_sg" {
  vpc_id = aws_vpc.wordpress_vpc.id
  ingress {
    from_port   = var.ingress_ports[0]
    to_port     = var.ingress_ports[0]
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = var.ingress_ports[1]
    to_port     = var.ingress_ports[1]
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = var.ingress_ports[2]
    to_port     = var.ingress_ports[2]
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = var.ingress_ports[3]
    to_port     = var.ingress_ports[3]
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
    Name = "wordpress-sg"
  }
}
resource "aws_instance" "wordpress_ec2" {
  ami           = var.ami_id  # Replace with the correct Amazon Linux 2 AMI ID
  instance_type = var.instance_type
  key_name      = "MyKeyPair"
  vpc_security_group_ids = [aws_security_group.wordpress_sg.id]
  subnet_id     = aws_subnet.public_subnet[0].id  # Place in public subnet 1
}
resource "aws_security_group" "rds_sg" {
  vpc_id = aws_vpc.wordpress_vpc.id
  ingress {
    from_port   = 3306
    to_port     = 3306
    protocol    = "tcp"
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "rds-sg"
  }
}
resource "aws_db_subnet_group" "wordpress_db_subnet_group" {
  name       = "wordpress-db-subnet-group"
  subnet_ids = values(aws_subnet.private_subnet)[*].id
}
resource "aws_db_instance" "mysql" {
  identifier            = "mysql"
  allocated_storage     = var.allocated_storage
  storage_type          = "gp2"
  instance_class        = var.db_instance_class
  engine                = "mysql"
  username              = var.db_username
  password              = var.db_password
  skip_final_snapshot   = true
# vpc_security_group_ids = [aws_security_group.rds_sg.id]
  db_subnet_group_name  = aws_db_subnet_group.wordpress_db_subnet_group.name
}








# variable 


variable "vpc_cidr_block" {
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}
variable "subnet_count" {
  description = "Number of subnets to create"
  default     = 3
}
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
variable "ingress_ports" {
  description = "List of ingress ports"
  type        = list(number)
  default     = [80, 443, 22, 3306]
}
variable "ami_id" {
  description = "AMI ID for EC2 instance"
  default     = "ami-0cf10cdf9fcd62d37"
}
variable "instance_type" {
  description = "Instance type for EC2 instance"
  default     = "t2.micro"
}
variable "key_name" {
  description = "Key pair name for EC2 instance"
  default     = "ssh-key"
}
variable "allocated_storage" {
  description = "Allocated storage for RDS"
  default     = 20
}
variable "db_instance_class" {
  description = "Instance class for RDS"
  default     = "db.t2.micro"
}
variable "db_username" {
  description = "Username for RDS"
  default     = "admin"
}
variable "db_password" {
  description = "Password for RDS"
  default     = "adminadmin"
}
