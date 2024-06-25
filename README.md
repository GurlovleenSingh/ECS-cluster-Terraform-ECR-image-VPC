# ECS-cluster-Terraform-ECR-image-VPC
Use terraform to deploy ECS cluster from ECR image using Fargate
CODE:

****************************************************************************
provider "aws" {
  region = var.aws_region
}

# Define variables
variable "aws_region" {
  description = "AWS region"
  type        = string
}

variable "ecs_cluster_name" {
  description = "Name of the ECS cluster"
  type        = string
}

variable "ecr_repository_name" {
  description = "Name of the existing ECR repository"
  type        = string
}

variable "image_tag" {
  description = "Tag of the ECR image to use"
  type        = string
}
variable "vpc_cidr" {
  type = string
  
}
resource "aws_vpc" "main" {
 cidr_block           = var.vpc_cidr
 enable_dns_hostnames = true
 tags = {
   name = "main"
 }
}
resource "aws_subnet" "subnet" {
 vpc_id                  = aws_vpc.main.id
 cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 1)
 map_public_ip_on_launch = true
 availability_zone       = "us-east-1a"
}
resource "aws_ecs_cluster" "cluster_name" {
  name = var.ecs_cluster_name
}
resource "aws_subnet" "subnet2" {
 vpc_id                  = aws_vpc.main.id
 cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 2)
 map_public_ip_on_launch = true
 availability_zone       = "us-east-1b"
}
resource "aws_internet_gateway" "internet_gateway" {
 vpc_id = aws_vpc.main.id
 tags = {
   Name = "internet_gateway"
 }
}
resource "aws_route_table" "route_table" {
 vpc_id = aws_vpc.main.id
 route {
   cidr_block = "0.0.0.0/0"
   gateway_id = aws_internet_gateway.internet_gateway.id
 }
}

resource "aws_route_table_association" "subnet_route" {
 subnet_id      = aws_subnet.subnet.id
 route_table_id = aws_route_table.route_table.id
}

resource "aws_route_table_association" "subnet2_route" {
 subnet_id      = aws_subnet.subnet2.id
 route_table_id = aws_route_table.route_table.id
}
resource "aws_security_group" "security_group" {
 name   = "ecs-security-group"
 vpc_id = aws_vpc.main.id

 ingress {
   from_port   = 0
   to_port     = 0
   protocol    = -1
   self        = "false"
   cidr_blocks = ["0.0.0.0/0"]
   description = "any"
 }

 egress {
   from_port   = 0
   to_port     = 0
   protocol    = "-1"
   cidr_blocks = ["0.0.0.0/0"]
 }
}

resource "aws_iam_role" "ecs_task_execution_role" {
  name               = "ecsTaskExecutionRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect    = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Action    = "sts:AssumeRole"
    }]
  })
}
# IAM Policy Attachment for ECS Task Execution Role
resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy_attachment" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
# ECS Task Definition
resource "aws_ecs_task_definition" "my_task_definition" {
  family                   = "my-ecs-task"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  network_mode             = "awsvpc"
  cpu                      = "256"     # CPU units for the task
  memory                   = "512"     # Memory for the task (in MiB)

  requires_compatibilities = ["FARGATE"]

  container_definitions = jsonencode([
    {
      name      = "my-container"
      image     = "your_ecr_image_uri"  # Replace with your ECR image URI
      cpu       = 256
      memory    = 512
      essential = true
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
          protocol      = "tcp"
        }
      ]
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "my_ecs_service" {
  name            = "my-ecs-service"
  cluster         = aws_ecs_cluster.cluster_name.id # Replace with your ECS cluster name
  task_definition = aws_ecs_task_definition.my_task_definition.arn
  desired_count   = 2
  network_configuration {
   subnets         = [aws_subnet.subnet.id, aws_subnet.subnet2.id]
   security_groups = [aws_security_group.security_group.id]
 }


  launch_type     = "FARGATE"  # Or "EC2" if using EC2 launch type

  depends_on = [aws_ecs_task_definition.my_task_definition]
}






Explanation of the Terraform Configuration

Provider Block: Specifies the AWS provider and the region where resources will be managed.

aws_iam_role Resource (ecs_task_execution_role): Defines an IAM role named "ecs-task-execution-role" that ECS will use to execute tasks. It allows ECS to assume this role (ecs-tasks.amazonaws.com service principal).

aws_iam_role_policy_attachment Resource (ecs_task_execution_policy_attachment): Attaches the AmazonECSTaskExecutionRolePolicy managed policy to the IAM role. This policy grants ECS the necessary permissions to manage tasks.

aws_vpc and aws_subnet Resources: Creates a VPC and subnets if not already defined. Adjust cidr_block, availability_zone, and other parameters as per your AWS environment.

aws_ecs_cluster Resource (my_cluster): Defines an ECS cluster named "my-ecs-cluster" where ECS tasks will be deployed.

aws_ecs_task_definition Resource (my_task_definition): Defines an ECS task definition named "my-ecs-task" that specifies the container configuration. Replace "your_ecr_image_uri" with the URI of your ECR image. Adjust CPU, memory, container ports, and other settings as per your application requirements.
