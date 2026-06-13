# Terraform_Learning



# 🏗️ Terraform: Beginner to Production — Mapped to AWS ECS

> **For Prem** — You provision AWS resources (ECS clusters, VPCs, ALBs) through the console or CLI. This course teaches Terraform from scratch to production-grade IaC, mapping every concept to the AWS infrastructure you already know.

---

## 🧭 Quick Mental Model: Terraform ↔ AWS Console/CLI

| Terraform Concept | AWS Equivalent | What It Means |
|---|---|---|
| Provider | AWS Console/CLI | Plugin that talks to AWS API |
| Resource | Individual AWS resource | EC2, VPC, ECS cluster, etc. |
| Data Source | AWS API read (describe-*) | Read existing AWS resources |
| Variable | CLI parameter | Input to your Terraform code |
| Output | CLI output / console value | Exported value from Terraform |
| State (`terraform.tfstate`) | AWS's internal state | Tracks what Terraform manages |
| Module | CloudFormation nested stack | Reusable package of resources |
| Workspace | Separate AWS account/env | Same code, different environments |
| Backend | S3 bucket for state | Where state is stored remotely |
| `terraform plan` | Dry run / preview | See what will change before applying |
| `terraform apply` | Actually creating resources | Make the changes happen |
| `terraform destroy` | Deleting resources | Tear everything down |
| Remote State Locking | — (prevent concurrent writes) | Prevent corrupt state |
| Provisioner | User data / bootstrap scripts | Run commands on resource creation |
| `for_each` | CloudFormation Fn::ForEach | Create multiple similar resources |
| `dynamic` block | CloudFormation dynamic ref | Generate nested blocks dynamically |
| Lifecycle rules | — (prevent destroy/replace) | Protect critical resources |
| Import | Adopt existing resource | Bring AWS resource under Terraform |
| Moved blocks | — (refactor safely) | Rename/move resources without destroy |

---

## 📦 Module 1: What Is Terraform & Why IaC

### 1.1 The Problem Without IaC

```
Manual Infrastructure (ClickOps):
┌──────────────────────────────────────────────┐
│  "Who created this security group?"            │
│  "What settings did we use for the ALB?"       │
│  "Staging and prod are configured differently" │
│  "We can't recreate this environment"          │
│  "Disaster recovery = start from scratch"      │
└──────────────────────────────────────────────┘

Infrastructure as Code (Terraform):
┌──────────────────────────────────────────────┐
│  ✅ Version controlled (Git)                   │
│  ✅ Reproducible (same code = same infra)     │
│  ✅ Reviewable (PR for infra changes)         │
│  ✅ Automated (CI/CD applies changes)         │
│  ✅ Documented (code IS the documentation)    │
│  ✅ Disaster recovery = terraform apply        │
└──────────────────────────────────────────────┘
```

### 1.2 Terraform vs CloudFormation

```
┌──────────────────┬──────────────────────────────────────┐
│ Terraform         │ CloudFormation                        │
├──────────────────┼──────────────────────────────────────┤
│ Multi-cloud       │ AWS only                              │
│ HCL language      │ YAML/JSON                             │
│ State file (S3)   │ State managed by AWS                 │
│ Plan before apply │ Change sets (similar)                 │
│ Modules           │ Nested stacks                         │
│ No rollback       │ Automatic rollback on failure          │
│ Import existing    │ Import existing (similar)              │
│ Huge community    │ AWS-native integration                │
└──────────────────┴──────────────────────────────────────┘
```

---

## 📦 Module 2: Terraform Basics

### 2.1 Install & Configure

```bash
# Install
brew install terraform

# Verify
terraform version

# Enable autocompletion
terraform -install-autocomplete
```

### 2.2 Provider Configuration

```hcl
# provider.tf
provider "aws" {
  region = "us-east-1"

  # Option 1: AWS CLI credentials (default)
  # Uses ~/.aws/credentials or env vars

  # Option 2: Assume role
  # assume_role {
  #   role_arn = "arn:aws:iam::123456789012:role/TerraformAdmin"
  # }

  # Option 3: Specific profile
  # profile = "production"
}

# For multiple AWS accounts
provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}
```

### 2.3 Your First Resource

```hcl
# main.tf
resource "aws_s3_bucket" "my_bucket" {
  bucket = "prem-my-app-bucket"

  tags = {
    Name        = "My App Bucket"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}
```

### 2.4 The Terraform Workflow

```bash
# 1. Initialize (download providers, modules)
terraform init

# 2. Format code (auto-format to standard style)
terraform fmt

# 3. Validate (check syntax)
terraform validate

# 4. Plan (preview changes — ALWAYS DO THIS)
terraform plan

# 5. Apply (make changes)
terraform apply              # Will show plan first
terraform apply -auto-approve # Skip confirmation (use in CI/CD only)

# 6. Destroy (tear down — DANGER!)
terraform destroy
```

### 2.5 What Happens Behind the Scenes

```
terraform apply:
  1. Read .tf files → Build dependency graph
  2. Refresh state → Compare with real infrastructure
  3. Calculate diff → What needs to be created/updated/deleted
  4. Show plan → "1 to add, 0 to change, 0 to destroy"
  5. Apply → Call AWS API to make changes
  6. Update state → Write new state to terraform.tfstate
```

---

## 📦 Module 3: Variables, Outputs & Data Sources

### 3.1 Variables (Inputs)

```hcl
# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid CIDR block."
  }
}

variable "public_subnets" {
  description = "Public subnet CIDRs"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium", "t3.large"], var.instance_type)
    error_message = "Instance type must be a valid t3 size."
  }
}
```

```bash
# Override variables
terraform apply -var="environment=production"
terraform apply -var-file="production.tfvars"
```

```hcl
# production.tfvars
environment = "production"
vpc_cidr    = "10.0.0.0/16"
instance_type = "t3.large"
```

### 3.2 Outputs (Exports)

```hcl
# outputs.tf
output "vpc_id" {
  description = "The VPC ID"
  value       = aws_vpc.main.id
}

output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.main.dns_name
}

output "ecs_cluster_name" {
  value = aws_ecs_cluster.main.name
}

# Sensitive output (hidden from logs)
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

### 3.3 Data Sources (Read Existing Resources)

```hcl
# Read current AWS account info
data "aws_caller_identity" "current" {}

# Read default VPC
data "aws_vpc" "default" {
  default = true
}

# Read latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Read subnets by tag
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [aws_vpc.main.id]
  }
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

# Read SSM parameter (latest ECS-optimized AMI)
data "aws_ssm_parameter" "ecs_optimized_ami" {
  name = "/aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id"
}

# Use data sources
resource "aws_launch_configuration" "ecs" {
  image_id = data.aws_ssm_parameter.ecs_optimized_ami.value
  # ...
}
```

---

## 📦 Module 4: State Management

### 4.1 What Is State?

State is Terraform's **source of truth** — it maps your `.tf` code to real AWS resources.

```
terraform.tfstate maps:
  aws_vpc.main → vpc-0abc123def456
  aws_subnet.public[0] → subnet-111
  aws_subnet.public[1] → subnet-222
  aws_ecs_cluster.main → arn:aws:ecs:us-east-1:123:cluster/my-cluster

Without state, Terraform can't know:
  - Which real resources belong to which code resources
  - Whether a resource already exists or needs creating
  - What the current configuration is
```

### 4.2 Remote State (S3 + DynamoDB) — PRODUCTION MUST

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "prem-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # State locking!
    kms_key_id     = "alias/terraform-key"
  }
}
```

```bash
# Create the S3 bucket and DynamoDB table FIRST (bootstrap)
aws s3 mb s3://prem-terraform-state
aws s3api put-bucket-versioning --bucket prem-terraform-state \
  --versioning-configuration Status=Enabled
aws dynamodb create-table --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### 4.3 State Commands

```bash
# List resources in state
terraform state list

# Show resource details
terraform state show aws_vpc.main

# Move resource (rename in state)
terraform state mv aws_vpc.main aws_vpc.primary

# Remove resource from state (keep in AWS!)
terraform state rm aws_vpc.main

# Import existing AWS resource into state
terraform import aws_vpc.main vpc-0abc123def456
```

### 4.4 State Locking

```
┌──────────────────────────────────────────────────────┐
│  Without locking:                                     │
│  Dev A: terraform apply → reads state                 │
│  Dev B: terraform apply → reads state (same version!) │
│  Dev A: writes state                                   │
│  Dev B: writes state → OVERWRITES Dev A's changes! 💥  │
│                                                       │
│  With DynamoDB locking:                               │
│  Dev A: terraform apply → acquires lock               │
│  Dev B: terraform apply → LOCKED, must wait          │
│  Dev A: completes, releases lock                      │
│  Dev B: acquires lock, proceeds safely ✅             │
└──────────────────────────────────────────────────────┘
```

---

## 📦 Module 5: Modules — Reusable Infrastructure

### 5.1 What Are Modules?

Modules are **reusable packages of Terraform code** — like functions in programming.

```
Module structure:
modules/
├── vpc/
│   ├── main.tf        # Resources
│   ├── variables.tf   # Inputs
│   ├── outputs.tf     # Outputs
│   └── versions.tf    # Provider requirements
├── ecs/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── alb/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

### 5.2 Creating a VPC Module

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnets)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Type = "public"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Type = "private"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name}-igw" })
}

resource "aws_nat_gateway" "this" {
  count         = length(var.public_subnets)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.this]
  tags          = merge(var.tags, { Name = "${var.name}-nat-${count.index + 1}" })
}

resource "aws_eip" "nat" {
  count  = length(var.public_subnets)
  domain = "vpc"
  tags   = merge(var.tags, { Name = "${var.name}-eip-${count.index + 1}" })
}
```

```hcl
# modules/vpc/variables.tf
variable "name"            { type = string }
variable "cidr"            { type = string }
variable "azs"             { type = list(string) }
variable "public_subnets"  { type = list(string) }
variable "private_subnets" { type = list(string) }
variable "tags"            { type = map(string); default = {} }
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id"         { value = aws_vpc.this.id }
output "public_subnet_ids"  { value = aws_subnet.public[*].id }
output "private_subnet_ids" { value = aws_subnet.private[*].id }
```

### 5.3 Using the Module

```hcl
# environments/production/main.tf
module "vpc" {
  source = "../../modules/vpc"

  name            = "prod"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.20.0/24"]
  tags            = local.common_tags
}
```

### 5.4 Using Public Registry Modules

```hcl
# Terraform Registry modules (community-maintained)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.20.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = true  # Cost saving for non-prod
}
```

---

## 📦 Module 6: ECS Infrastructure with Terraform

### 6.1 Complete ECS Fargate Setup

```hcl
# ecs-cluster.tf
resource "aws_ecs_cluster" "main" {
  name = "${var.name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = local.common_tags
}

# IAM Role for ECS Task Execution
resource "aws_iam_role" "ecs_task_execution" {
  name = "${var.name}-ecs-task-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Definition
resource "aws_ecs_task_definition" "api" {
  family                   = "${var.name}-api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "api"
      image     = "${var.ecr_repo_url}:latest"
      essential = true
      portMappings = [{
        containerPort = 3000
        protocol     = "tcp"
      }]
      environment = [
        { name = "NODE_ENV", value = var.environment }
      ]
      secrets = [
        { name = "DB_PASSWORD", valueFrom = var.db_password_arn }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = "/ecs/${var.name}-api"
          "awslogs-region"        = var.region
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])

  tags = local.common_tags
}

# ECS Service
resource "aws_ecs_service" "api" {
  name            = "${var.name}-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = module.vpc.private_subnet_ids
    security_groups = [aws_security_group.api.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api"
    container_port   = 3000
  }

  deployment_controller {
    type = "ECS"  # Or "CODE_DEPLOY" for blue/green
  }

  lifecycle {
    ignore_changes = [desired_count]  # HPA manages count
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "api" {
  max_capacity       = var.max_count
  min_capacity       = var.min_count
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "${var.name}-api-cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.api.resource_id
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  service_namespace  = aws_appautoscaling_target.api.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification { predefined_metric_type = "ECSServiceAverageCPUUtilization" }
    target_value       = 70
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### 6.2 ALB + Security Groups

```hcl
# alb.tf
resource "aws_lb" "main" {
  name               = "${var.name}-alb"
  internal           = false
  load_balancer_type = "application"
  subnets            = module.vpc.public_subnet_ids
  security_groups    = [aws_security_group.alb.id]

  tags = local.common_tags
}

resource "aws_lb_target_group" "api" {
  name        = "${var.name}-api-tg"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# Security Groups
resource "aws_security_group" "alb" {
  name   = "${var.name}-alb-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_security_group" "api" {
  name   = "${var.name}-api-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Only ALB can reach API
  }
  egress { from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"] }
}
```

---

## 📦 Module 7: Advanced Terraform — Loops, Conditionals & Dynamics

### 7.1 `count` — Create Multiple Resources

```hcl
# Create multiple subnets
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "private" {
  count             = length(var.subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = { Name = "private-${count.index + 1}" }
}
```

### 7.2 `for_each` — Create from Map (More Flexible)

```hcl
variable "services" {
  default = {
    api = { port = 3000, cpu = 512, memory = 1024 }
    web = { port = 80,   cpu = 256, memory = 512  }
    worker = { port = 0, cpu = 256, memory = 512 }
  }
}

resource "aws_ecs_task_definition" "service" {
  for_each = var.services

  family                   = "${var.name}-${each.key}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = each.value.cpu
  memory                   = each.value.memory
  # ...
}
```

### 7.3 `dynamic` Blocks

```hcl
# Dynamic blocks for variable nested config
resource "aws_ecs_task_definition" "api" {
  # ...

  container_definitions = jsonencode([{
    name = "api"
    # Dynamic environment variables
    environment = [for k, v in var.env_vars : { name = k, value = v }]

    # Dynamic secrets
    secrets = [for k, v in var.secret_arns : { name = k, valueFrom = v }]
  }])
}
```

### 7.4 Conditionals

```hcl
# Create resource only in production
resource "aws_nat_gateway" "this" {
  count = var.environment == "production" ? 2 : 1
  # ...
}

# Enable/disable feature
resource "aws_ecs_service" "api" {
  count = var.enable_api ? 1 : 0
  # ...
}

# Ternary operator
# condition ? true_value : false_value
```

### 7.5 `locals` — Computed Values

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = "prem"
  }

  # Merge tags with resource-specific tags
  api_tags = merge(local.common_tags, { Service = "api" })
}
```

---

## 📦 Module 8: Workspaces & Environment Management

### 8.1 Workspace Strategy

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new staging
terraform workspace new production

# Switch workspace
terraform workspace select production

# Show current
terraform workspace show
```

```hcl
# Use workspace for environment-specific config
variable "env_config" {
  type = map(object({
    instance_type = string
    min_count     = number
    max_count     = number
  }))
  default = {
    staging = { instance_type = "t3.small", min_count = 1, max_count = 3 }
    production = { instance_type = "t3.large", min_count = 2, max_count = 10 }
  }
}

locals {
  config = var.env_config[terraform.workspace]
}

resource "aws_ecs_service" "api" {
  desired_count = local.config.min_count
  # ...
}
```

### 8.2 Directory-Based Environments (RECOMMENDED for Production)

```
├── modules/                    # Shared modules
│   ├── vpc/
│   ├── ecs/
│   └── alb/
├── environments/
│   ├── staging/
│   │   ├── main.tf            # Uses modules with staging values
│   │   ├── variables.tf
│   │   ├── staging.tfvars
│   │   └── backend.tf         # State: staging/terraform.tfstate
│   └── production/
│       ├── main.tf             # Uses modules with production values
│       ├── variables.tf
│       ├── production.tfvars
│       └── backend.tf          # State: production/terraform.tfstate
```

---

## 📦 Module 9: Production Best Practices

### 9.1 Lifecycle Rules

```hcl
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    # Prevent accidental destruction of database!
    prevent_destroy = true

    # Ignore changes managed outside Terraform (e.g., auto-scaling)
    ignore_changes = [allocated_storage]

    # Create new before destroying old (zero-downtime)
    create_before_destroy = true
  }
}
```

### 9.2 Moved Blocks (Safe Refactoring)

```hcl
# When you rename a resource, use moved to prevent destroy+recreate
moved {
  from = aws_ecs_cluster.main
  to   = aws_ecs_cluster.primary
}
```

### 9.3 Import Existing Resources

```bash
# Bring existing AWS resource under Terraform management
terraform import aws_vpc.main vpc-0abc123
terraform import aws_ecs_cluster.main my-cluster
terraform import aws_s3_bucket.my_bucket my-existing-bucket
```

### 9.4 Version Constraints

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"  # Pin Terraform version

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allow 5.x but not 6.0
    }
  }
}

# Version constraint syntax:
# "~> 5.0"  → 5.0 to 5.99 (most common)
# ">= 5.0"  → 5.0 or higher
# "= 5.10"  → Exactly 5.10
# ">= 5.0, < 6.0" → Between 5.0 and 5.99
```

### 9.5 Sensitive Data Handling

```hcl
# Mark variables as sensitive
variable "db_password" {
  type      = string
  sensitive = true  # Hidden from plan output
}

# Mark outputs as sensitive
output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false
}

# Use AWS Secrets Manager for secrets
data "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}
```

### 9.6 Production Checklist

```
✅ Remote state (S3 + DynamoDB locking)
✅ State encryption (KMS)
✅ Version pinning (terraform + providers)
✅ Separate state per environment
✅ Modules for reusable code
✅ Variables with validation
✅ Sensitive data marked
✅ Lifecycle prevent_destroy on critical resources
✅ CI/CD for terraform plan/apply
✅ PR review for infrastructure changes
✅ Cost estimation (Infracost)
✅ Security scanning (tfsec, checkov)
✅ State backup (S3 versioning)
✅ Import existing resources before creating duplicates
```

---

## 📦 Module 10: Terraform in CI/CD

### 10.1 GitHub Actions Workflow

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
  push:
    branches: [main]

jobs:
  plan:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.7.0" }
      - run: terraform init -backend=false
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -var-file=production.tfvars

  apply:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/TerraformRole
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -var-file=production.tfvars -out=plan.tfplan
      - run: terraform apply plan.tfplan
```

---

## 🎯 Interview Questions — Terraform

### Beginner
1. What is Terraform and how does it differ from CloudFormation?
2. Explain the Terraform workflow (init → plan → apply).
3. What is Terraform state and why is it important?
4. What is the difference between `count` and `for_each`?
5. How do you manage different environments (dev, staging, prod)?

### Advanced
6. How does remote state work and why is state locking important?
7. Explain Terraform modules and when you'd create custom modules.
8. How would you import existing AWS resources into Terraform?
9. What are lifecycle rules and when would you use `prevent_destroy`?
10. How would you structure a production Terraform repository?

### Scenario-Based
11. Your `terraform apply` failed halfway. What's the state of your infrastructure?
12. A teammate manually changed an AWS resource. How do you fix the drift?
13. Design a Terraform structure for a microservice with VPC, ECS, ALB, and RDS.
14. How would you implement zero-downtime deployments with Terraform + ECS?
15. Your state file is corrupted. How do you recover?

---

## 📚 Recommended Learning Path

| Week | Focus | What to Do |
|------|-------|-----------|
| **1** | Basics | Install, provider, resources, plan/apply (Modules 1-2) |
| **2** | Core | Variables, outputs, data sources, state (Modules 3-4) |
| **3** | Modules | Create & use modules, ECS infrastructure (Modules 5-6) |
| **4** | Advanced | Loops, conditionals, workspaces (Modules 7-8) |
| **5** | Production | Best practices, CI/CD, interview prep (Modules 9-10) |

---

## 🔑 The One-Liner

> **Terraform turns your infrastructure into code — version it, review it, automate it.** If you can click it in the AWS console, you can write it in Terraform.

---

*Built with ❤️ by Cline SR for Prem — Terraform from beginner to production*
