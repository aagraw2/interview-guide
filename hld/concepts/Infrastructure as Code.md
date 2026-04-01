## 1. What is Infrastructure as Code (IaC)?

Infrastructure as Code is the practice of managing and provisioning infrastructure through machine-readable definition files rather than manual configuration.

```
Manual Infrastructure:
  - Click through AWS console
  - Run commands manually
  - Hard to reproduce
  - No version control
  - Error-prone

Infrastructure as Code:
  - Define in code (Terraform, CloudFormation)
  - Version controlled (git)
  - Reproducible
  - Automated
  - Testable
```

**Key principle:** Treat infrastructure like software — version controlled, tested, and automated.

---

## 2. Benefits

### Reproducibility

```
Same code → Same infrastructure

Deploy to multiple environments:
  - Development
  - Staging
  - Production

All identical (except config)
```

### Version Control

```
Infrastructure changes in git:
  - Review (pull requests)
  - Audit trail (git history)
  - Rollback (git revert)
  - Collaboration (branches)
```

### Automation

```
No manual steps:
  - Run terraform apply
  - Infrastructure created
  - Consistent, fast, reliable
```

---

## 3. IaC Tools

### Terraform

```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-server"
  }
}

resource "aws_security_group" "web_sg" {
  name = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### AWS CloudFormation

```yaml
# template.yaml
Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0c55b159cbfafe1f0
      InstanceType: t2.micro
      Tags:
        - Key: Name
          Value: web-server
  
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: web-sg
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
```

### Pulumi

```python
# __main__.py
import pulumi
import pulumi_aws as aws

# Create EC2 instance
web_server = aws.ec2.Instance(
    "web-server",
    ami="ami-0c55b159cbfafe1f0",
    instance_type="t2.micro",
    tags={"Name": "web-server"}
)

# Create security group
web_sg = aws.ec2.SecurityGroup(
    "web-sg",
    ingress=[{
        "protocol": "tcp",
        "from_port": 80,
        "to_port": 80,
        "cidr_blocks": ["0.0.0.0/0"]
    }]
)

pulumi.export("public_ip", web_server.public_ip)
```

---

## 4. Terraform Workflow

### Initialize

```bash
# Initialize Terraform
terraform init

# Downloads providers (AWS, GCP, etc.)
# Creates .terraform directory
```

### Plan

```bash
# Preview changes
terraform plan

# Shows:
#   + Resources to create
#   ~ Resources to modify
#   - Resources to delete
```

### Apply

```bash
# Apply changes
terraform apply

# Creates/updates infrastructure
# Saves state to terraform.tfstate
```

### Destroy

```bash
# Destroy infrastructure
terraform destroy

# Deletes all resources
```

---

## 5. State Management

### Terraform State

```
terraform.tfstate:
  - Current infrastructure state
  - Maps code to real resources
  - Critical (don't lose it!)

Local state:
  terraform.tfstate in directory
  
  Problems:
    - Not shared (team can't collaborate)
    - Not backed up (can be lost)
    - No locking (concurrent changes conflict)
```

### Remote State

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    
    # State locking
    dynamodb_table = "terraform-locks"
  }
}

Benefits:
  ✅ Shared (team collaboration)
  ✅ Backed up (S3 versioning)
  ✅ Locked (prevents conflicts)
```

---

## 6. Modules and Reusability

### Terraform Modules

```hcl
# modules/web-server/main.tf
variable "instance_type" {
  default = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

# main.tf (use module)
module "web_server" {
  source        = "./modules/web-server"
  instance_type = "t2.small"
}

output "web_ip" {
  value = module.web_server.public_ip
}
```

---

## 7. Environment Management

### Workspaces

```bash
# Create workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select prod

# Each workspace has separate state
```

### Directory Structure

```
infrastructure/
  environments/
    dev/
      main.tf
      variables.tf
      terraform.tfvars
    staging/
      main.tf
      variables.tf
      terraform.tfvars
    prod/
      main.tf
      variables.tf
      terraform.tfvars
  modules/
    web-server/
    database/
    network/
```

---

## 8. Best Practices

### Idempotency

```
Run terraform apply multiple times:
  - Same result each time
  - No duplicate resources
  - Safe to re-run

Terraform ensures idempotency
```

### Immutable Infrastructure

```
Don't modify running resources:
  ❌ SSH into server and update
  ✅ Update code, terraform apply, replace server

Benefits:
  - Consistent
  - Reproducible
  - No configuration drift
```

### Version Control

```
Store in git:
  ✅ *.tf files
  ✅ *.tfvars files (non-sensitive)
  ❌ terraform.tfstate (use remote state)
  ❌ .terraform/ directory
  ❌ Secrets (use variables or secrets manager)

.gitignore:
  .terraform/
  *.tfstate
  *.tfstate.backup
  *.tfvars (if contains secrets)
```

---

## 9. Testing Infrastructure

### Validation

```bash
# Validate syntax
terraform validate

# Format code
terraform fmt

# Lint (with tflint)
tflint
```

### Plan Review

```bash
# Review changes before apply
terraform plan

# Save plan
terraform plan -out=tfplan

# Apply saved plan
terraform apply tfplan
```

### Automated Testing

```python
# test_infrastructure.py (using pytest + boto3)
import boto3
import pytest

def test_web_server_exists():
    ec2 = boto3.client('ec2', region_name='us-east-1')
    response = ec2.describe_instances(
        Filters=[{'Name': 'tag:Name', 'Values': ['web-server']}]
    )
    assert len(response['Reservations']) > 0

def test_security_group_allows_http():
    ec2 = boto3.client('ec2', region_name='us-east-1')
    response = ec2.describe_security_groups(
        GroupNames=['web-sg']
    )
    sg = response['SecurityGroups'][0]
    
    http_rule = next(
        (rule for rule in sg['IpPermissions'] 
         if rule['FromPort'] == 80),
        None
    )
    assert http_rule is not None
```

---

## 10. Common Interview Questions + Answers

### Q: What is Infrastructure as Code and why is it important?

> "Infrastructure as Code is managing infrastructure through code rather than manual configuration. You define your infrastructure in files like Terraform or CloudFormation, version control them in git, and apply them automatically. It's important because it makes infrastructure reproducible — you can deploy identical environments consistently. It enables version control for infrastructure changes with review and rollback. It's automated, reducing human error. And it's testable, allowing you to validate infrastructure before deploying. This is essential for modern cloud-native applications."

### Q: What's the difference between Terraform and CloudFormation?

> "Terraform is cloud-agnostic and works with AWS, GCP, Azure, and many other providers using a single language (HCL). CloudFormation is AWS-specific and uses YAML or JSON. Terraform has better state management with remote backends and locking. CloudFormation integrates deeply with AWS services and is fully managed. Terraform requires running terraform apply, while CloudFormation can be triggered by AWS services. For multi-cloud or complex state management, use Terraform. For AWS-only with deep integration, CloudFormation works well."

### Q: How do you manage Terraform state?

> "Never use local state in production — it's not shared, not backed up, and has no locking. Use remote state with S3 backend for storage and DynamoDB for locking. This allows team collaboration, prevents concurrent modifications, and provides backup with S3 versioning. Store state separately per environment using different S3 keys or buckets. Never commit state files to git. Enable encryption at rest for sensitive data. And use state locking to prevent race conditions when multiple people run terraform apply."

### Q: What are the best practices for Infrastructure as Code?

> "First, version control everything except state and secrets. Second, use modules for reusability — don't repeat yourself. Third, separate environments with workspaces or directories. Fourth, use remote state with locking for collaboration. Fifth, always run terraform plan before apply to review changes. Sixth, make infrastructure immutable — replace rather than modify. Seventh, validate and test infrastructure code. Finally, use CI/CD to automate infrastructure deployment with proper review and approval processes."

---

## 11. Quick Reference

```
What is IaC?
  Manage infrastructure through code
  Version controlled, automated, reproducible

Benefits:
  - Reproducibility (same code → same infra)
  - Version control (git history, rollback)
  - Automation (no manual steps)
  - Testing (validate before deploy)

Popular tools:
  - Terraform (cloud-agnostic, HCL)
  - CloudFormation (AWS-specific, YAML/JSON)
  - Pulumi (multi-cloud, real programming languages)

Terraform workflow:
  1. terraform init (download providers)
  2. terraform plan (preview changes)
  3. terraform apply (create/update)
  4. terraform destroy (delete)

State management:
  - Local: terraform.tfstate (don't use in prod)
  - Remote: S3 + DynamoDB (shared, locked, backed up)

Modules:
  Reusable infrastructure components
  Define once, use multiple times

Environment management:
  - Workspaces (separate state per environment)
  - Directories (separate code per environment)

Best practices:
  - Version control (except state, secrets)
  - Remote state with locking
  - Modules for reusability
  - Immutable infrastructure
  - Plan before apply
  - Automated testing
  - CI/CD for deployment

Testing:
  - terraform validate (syntax)
  - terraform plan (preview)
  - Automated tests (pytest + boto3)
```
