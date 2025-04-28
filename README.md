# Automated-Terraform-deployment-pipeline-with-security-and-compliance-checks-on-AWS

---

# 🚀 AWS Terraform CI/CD Deployment with CodePipeline

---

## 📜 Project Objective

Build a **fully automated** CI/CD pipeline for **Terraform** using:

- **AWS ECR** for custom Docker image (Terraform + TFLint + Checkov + TFSec)
- **AWS CodeCommit** for infrastructure code storage
- **AWS CodeBuild** for validating, scanning, and deploying Terraform code
- **AWS CodePipeline** for orchestrating the deployment
- **AWS S3** and **KMS** for secure remote Terraform backend
- **Security and Compliance Scans** to prevent misconfigurations

---

# 📂 Project Structure

```bash
aws-codepipeline-terraform/
├── buildspec.yml              # CodeBuild instructions
├── backend.tf                  # Terraform backend config
├── codebuild.tf                # New! Full CodeBuild project
├── codecommit.tf               # CodeCommit repo
├── ecr.tf                      # New! Create ECR repository
├── iam.tf                      # New! Create CodeBuild IAM Role and Policies
├── kms.tf                      # KMS Key for backend and artifacts
├── pipeline.tf                 # CodePipeline definition
├── providers.tf                # AWS Provider config
├── sns.tf                      # SNS Topic for pipeline notifications
├── variables.tf                # Terraform input variables
├── terraform.tfvars            # Terraform values
├── docker/
│   ├── Dockerfile              # New! Custom Dockerfile
└── README.md

```


---
## file-setup.py

```python
import os

# Define base directory
base_dir = "/home/lilia/VIDEOS/aws-codepipeline-terraform"

# Define files and their contents
files = {
    "buildspec.yml": '''version: 0.2

phases:
  pre_build:
    commands:
      - terraform init

  build:
    commands:
      - terraform validate
      - tflint
      - checkov -d .
      - tfsec .

artifacts:
  files:
    - '**/*'
''',

    "backend.tf": '''terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "pipeline/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "<kms-key-id>"
  }
}
''',

    "codebuild.tf": '''resource "aws_codebuild_project" "terraform_pipeline" {
  name          = "terraform-ci-cd"

  environment {
    compute_type                = "BUILD_GENERAL1_SMALL"
    image                       = "${aws_account_id}.dkr.ecr.${var.region}.amazonaws.com/terraform-pipeline-image:latest"
    type                        = "LINUX_CONTAINER"
    image_pull_credentials_type = "SERVICE_ROLE"
    privileged_mode             = true
  }

  service_role = aws_iam_role.codebuild_service_role.arn

  source {
    type      = "CODECOMMIT"
    location  = aws_codecommit_repository.terraform_repo.clone_url_http
    buildspec = "buildspec.yml"
  }

  artifacts {
    type = "CODEPIPELINE"
  }
}
''',

    "codecommit.tf": '''resource "aws_codecommit_repository" "terraform_repo" {
  repository_name = var.app_name
  description     = "Terraform Infrastructure Repo"
}
''',

    "ecr.tf": '''resource "aws_ecr_repository" "terraform_pipeline_image" {
  name = "terraform-pipeline-image"

  image_scanning_configuration {
    scan_on_push = true
  }

  encryption_configuration {
    encryption_type = "AES256"
  }
}
''',

    "iam.tf": '''resource "aws_iam_role" "codebuild_service_role" {
  name = "codebuild-terraform-service-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "codebuild.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "codebuild_policy_attach" {
  role       = aws_iam_role.codebuild_service_role.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}
''',

    "kms.tf": '''resource "aws_kms_key" "terraform_key" {
  description = "KMS key for encrypting Terraform backend and CodePipeline artifacts"
}
''',

    "pipeline.tf": '''resource "aws_codepipeline" "terraform_pipeline" {
  name     = "terraform-pipeline"
  role_arn = aws_iam_role.codepipeline_service_role.arn

  artifact_store {
    location = "my-terraform-state-bucket"
    type     = "S3"
    encryption_key {
      id   = aws_kms_key.terraform_key.arn
      type = "KMS"
    }
  }

  stage {
    name = "Source"

    action {
      name             = "SourceAction"
      category         = "Source"
      owner            = "AWS"
      provider         = "CodeCommit"
      version          = "1"
      output_artifacts = ["source_output"]

      configuration = {
        RepositoryName = aws_codecommit_repository.terraform_repo.name
        BranchName     = "main"
      }
    }
  }

  stage {
    name = "Build"

    action {
      name             = "BuildAction"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]
      version          = "1"

      configuration = {
        ProjectName = aws_codebuild_project.terraform_pipeline.name
      }
    }
  }
}
''',

    "providers.tf": '''provider "aws" {
  region = var.region
}
''',

    "sns.tf": '''resource "aws_sns_topic" "pipeline_notifications" {
  name = var.sns_topic_name
}
''',

    "variables.tf": '''variable "region" {
  description = "AWS Region"
  type        = string
  default     = "us-east-1"
}

variable "sns_topic_name" {
  description = "SNS Topic name for pipeline notifications"
  type        = string
}

variable "app_name" {
  description = "Application name"
  type        = string
}
''',

    "terraform.tfvars": '''region          = "us-east-1"
sns_topic_name = "terraform-pipeline-topic"
app_name       = "terraform-infra-app"
'''
}

# Create Dockerfile separately inside docker/ folder
dockerfile_path = os.path.join(base_dir, "docker")
os.makedirs(dockerfile_path, exist_ok=True)
with open(os.path.join(dockerfile_path, "Dockerfile"), "w") as f:
    f.write('''FROM amazonlinux:2

RUN yum update -y && \
    yum install -y unzip wget curl git python3 pip yum-utils sudo

RUN yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo && \
    yum install -y terraform

RUN curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

RUN pip3 install checkov

RUN curl -L https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 -o /usr/local/bin/tfsec && chmod +x /usr/local/bin/tfsec

RUN terraform -version && tflint --version && checkov --version && tfsec --version

CMD ["/bin/bash"]
''')

# Create files
os.makedirs(base_dir, exist_ok=True)
for filename, content in files.items():
    path = os.path.join(base_dir, filename)
    with open(path, "w") as f:
        f.write(content)

print("✅ All files created successfully!")

```


---

# 🛠️ Step-by-Step Implementation

---

## 1️⃣ Build and Push the Custom Docker Image

### Dockerfile

```Dockerfile
# Use an official minimal Linux base image
FROM amazonlinux:2

# Install dependencies
RUN yum update -y && \
    yum install -y \
    unzip \
    wget \
    curl \
    git \
    python3 \
    pip \
    yum-utils \
    sudo

# Install Terraform
RUN yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo && \
    yum -y install terraform

# Install TFLint
RUN curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Install Checkov
RUN pip3 install checkov

# Install TFSec
RUN curl -L https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 -o /usr/local/bin/tfsec && \
    chmod +x /usr/local/bin/tfsec

# Display installed versions for verification
RUN terraform -version && \
    tflint --version && \
    checkov --version && \
    tfsec --version

# Default command : Opens Bash shell when container starts
CMD ["/bin/bash"]


```
---

### Why?
Create an image that has `terraform`, `tflint`, `checkov`, and `tfsec` installed, to use in CodeBuild.

---

### Commands:

```bash
# 1. Create ECR Repository
aws ecr create-repository --repository-name terraform-pipeline-image

# 2. Authenticate Docker with AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com

# 3. Build the Docker image
docker build -t terraform-pipeline-image .

# 4. Tag the Docker image
docker tag terraform-pipeline-image:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/terraform-pipeline-image:latest

# 5. Push the image
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/terraform-pipeline-image:latest
```



🛠 How to Reference This Image in CodeBuild
In your CodeBuild project (in Terraform or Console UI), under Environment → Image, select:
```bash
aws_account_id.dkr.ecr.us-east-1.amazonaws.com/terraform-pipeline-image:latest

```

---

## 2️⃣ Set up CodeCommit and CodeBuild with Terraform

---

### Why?
- **CodeCommit**: Stores our Terraform infrastructure code.
- **CodeBuild**: Executes terraform init, validate, TFLint, Checkov, and TFSec.

---

### Files:

- `codecommit.tf`  #	Creates a Git repository (CodeCommit) to store your Terraform code.
- `codebuild.tf`   # Creates a build project (CodeBuild) that runs Terraform validation, linting, and security scans using 
                    your custom Docker image.
- `buildspec.yml`  # Defines the build steps CodeBuild should execute (like terraform init, terraform validate, tflint, 
                       checkov, tfsec).
- `iam.tf`         # Creates IAM roles and permissions needed for CodeBuild (and optionally CodePipeline) to securely                         pull images, access S3 backend, and deploy resources.
---

### Example `buildspec.yml`:

```yaml
version: 0.2

phases:
  install:   #Install any missing tools before doing anything.
    commands:
      - echo Installing security scanning tools...
      - curl -L https://github.com/tfsec/tfsec/releases/latest/download/tfsec-linux-amd64 -o /usr/local/bin/tfsec
      - chmod +x /usr/local/bin/tfsec

  pre_build:  #Prepare your project.Without terraform init, Terraform cannot do anything!
    commands:
      - terraform init

  build:   #Actually validate, lint, and scan the Terraform code.
    commands:
      - terraform validate
      - tflint
      - checkov -d .
      - tfsec .

artifacts:  #Collect any output files (optional). Upload everything (**/*) from the workspace as artifacts.
  files:
    - '**/*'
```

---

## 3️⃣ Set up CodePipeline with Terraform

---

### Why?
Automate the CI/CD flow with stages:

- **Source**: Pull from CodeCommit
- **Validate**: Terraform validate
- **Scan**: TFLint, Checkov, TFSec
- **Approval**: Manual approval (SNS notification optional)
- **Apply**: Terraform apply

---

### File:

- `pipeline.tf`

✅ Add manual approval stage to prevent automatic deployments to production without verification.

---

## 4️⃣ Set up Remote Backend (S3 + KMS)

---

### Why?
- Store Terraform state securely.
- Encrypt data with KMS.
- Enable state versioning.

---

### Commands:

```bash
# 1. Create S3 bucket
aws s3api create-bucket --bucket my-terraform-state-bucket --region us-east-1

# 2. Enable versioning
aws s3api put-bucket-versioning --bucket my-terraform-state-bucket --versioning-configuration Status=Enabled

# 3. Create KMS key
aws kms create-key --description "KMS key for Terraform state and CodePipeline artifacts"
```

---

### Example `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "pipeline/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "<kms-key-id>"
  }
}
```

---

## 5️⃣ Deploy and Test via the Pipeline

---

### 5a. Deploy a Non-Compliant (BAD) Infrastructure (Expect Failure)

---

### Example: `bad-s3-bucket.tf`

```hcl
resource "aws_s3_bucket" "bad_bucket" {
  bucket = "bad-bucket-example-123456"
  acl    = "public-read"  # ❌ Public access

  tags = {
    Name        = "BadBucket"
    Environment = "Dev"
  }
}

resource "aws_s3_bucket_public_access_block" "bad_bucket_block" {
  bucket = aws_s3_bucket.bad_bucket.id

  block_public_acls   = false
  block_public_policy = false
  ignore_public_acls  = false
  restrict_public_buckets = false
}
```

---

### ❌ Problems:

| Issue | Reason |
|:------|:-------|
| `acl = "public-read"` | Makes bucket public |
| `block_public_acls = false` | Allows dangerous ACLs |
| No encryption | No server-side security |

✅ **TFLint**, **Checkov**, and **TFSec** will catch and fail the pipeline at the scan phase.

---

### Expected Failure Output:

```bash
Checkov failed: S3 bucket bad_bucket allows public access (CKV_AWS_18)
Checkov failed: S3 bucket bad_bucket does not have encryption enabled (CKV_AWS_19)
```

---

### Deploy it:

```bash
git add .
git commit -m "Add non-compliant bad S3 bucket"
git push origin main
```

Pipeline will fail before `terraform apply`.

---

### 5b. Deploy a Compliant (GOOD) Infrastructure (Expect Success)

---

✅ After testing failure, replace bad S3 with good compliant S3 configuration and trigger the pipeline again.

```hcl
resource "aws_s3_bucket" "good_bucket" {
  bucket = "good-compliant-bucket-123456"  # Must be globally unique
  acl    = "private"

  tags = {
    Name        = "GoodBucket"
    Environment = "Production"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "good_bucket_encryption" {
  bucket = aws_s3_bucket.good_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_key.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "good_bucket_block" {
  bucket = aws_s3_bucket.good_bucket.id

  block_public_acls   = true
  block_public_policy = true
  ignore_public_acls  = true
  restrict_public_buckets = true
}


```
🚀 Triggering the Pipeline
After updating your Terraform files:
```bash
git add .
git commit -m "Replace bad S3 bucket with compliant configuration"
git push origin main

```
✅ This triggers the CodePipeline automatically.
✅ The new pipeline run should:
 - Pass terraform validate
 - Pass TFLint checks
 - Pass Checkov compliance checks
 - Pass TFSec security scans
✅ After passing, the infrastructure is safely deployed through Terraform.

---

# 🔎 Security and Compliance Tools

| Tool | Purpose |
|:-----|:--------|
| **TFLint** | Terraform syntax, style, and best practices |
| **Checkov** | Static analysis for security/compliance violations |
| **TFSec** | Security-focused static analysis |

✅ All scans happen automatically after Terraform validate.

✅ If violations are detected, the pipeline stops and does not proceed to deploy.

---

# ⚡ Quick CLI Summary

```bash
# 1. Create ECR
aws ecr create-repository --repository-name terraform-pipeline-image

# 2. Build and Push Docker Image
docker build -t terraform-pipeline-image .
docker tag terraform-pipeline-image:latest <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/terraform-pipeline-image:latest
docker push <aws_account_id>.dkr.ecr.us-east-1.amazonaws.com/terraform-pipeline-image:latest

# 3. Create S3 Bucket and Enable Versioning
aws s3api create-bucket --bucket my-terraform-state-bucket --region us-east-1
aws s3api put-bucket-versioning --bucket my-terraform-state-bucket --versioning-configuration Status=Enabled

# 4. Create KMS Key
aws kms create-key --description "KMS key for Terraform state and pipeline artifacts"

# 5. Initialize and Apply Terraform
terraform init
terraform plan
terraform apply
```

---

# 🎯 Final Flow Overview

```
Developer pushes code → 
CodeCommit → 
CodePipeline → 
CodeBuild (Validate + Scan) → 
Manual Approval → 
Terraform Apply (Deploy infrastructure)
```

---

# 🧠 Key Takeaways

- Always **scan code before applying** infrastructure
- **Catch mistakes early** (S3 public access, missing encryption, insecure IAM)
- **Automate security** into your pipeline
- Easily extend to ECS, EKS, RDS, VPC, etc.

---


# 📢 Important!

✅ Create ECR repo first  
✅ Push Terraform custom image  
✅ Make sure CodeBuild service role has:
 - Pull access to ECR (AmazonEC2ContainerRegistryReadOnly)
 - Administrator (or specific Terraform resource) access for deployments.
✅ Your ECR image must be pushed before you apply Terraform the first time.
✅ Set up backend (S3 + KMS)  
✅ Deploy intentionally bad infra → fail → fix → succeed ✅

---

# Final Result
 - You control everything — no external images
 - Faster builds — because tools are pre-installed
 - Security enforced — Checkov, TFSec, TFLint run before any deployment
 - Modern production-ready infrastructure pipeline!



---

 **visual diagram** for this project   
It would show visually: `CodeCommit ➔ CodePipeline ➔ CodeBuild ➔ Approval ➔ Deploy`.  

---

---

---

# 🌍 Real-World Scenario:  
**Secure Infrastructure Deployment for a Growing SaaS Company**

---

## 🏢 Company Context:

You work as a **DevOps Engineer** at a **Software-as-a-Service (SaaS) company** that builds **cloud-native applications** for global clients.  
The company is **rapidly scaling**, and now infrastructure needs to be:

- **Automated**
- **Consistent**
- **Secure**
- **Auditable**
- **Fast to deploy**

Manual `terraform apply` from laptops is **no longer acceptable** because of:

- Human error risks
- Lack of compliance checks
- Inconsistent deployments

---

## 🛠️ What You Build Using This Project:

You set up a **CI/CD pipeline on AWS** where:

1. **Developers** or **Infrastructure Engineers** write Terraform code and **push it** to a **CodeCommit repository**.  
   (Examples: Create new VPCs, EC2 instances, S3 buckets, EKS clusters.)

2. **AWS CodePipeline** automatically triggers when new code is pushed:

   - **AWS CodeBuild** runs several checks:
     - `terraform validate` (syntax validation)
     - `tflint` (best practices and linting)
     - `checkov` (compliance scanning for standards like PCI, SOC2, HIPAA)
     - `tfsec` (deep security scanning: encryption, IAM misconfigurations)

3. If the code **passes validation and security scans**:

   - It either:
     - **Moves directly to Terraform plan and apply**, deploying automatically  
     **OR**
     - **Waits for manual approval** (if an approval stage is configured).

4. Once approved, **Terraform automatically deploys** the infrastructure securely and consistently.

---

✅ This ensures that **only safe, compliant, and validated infrastructure** gets deployed into production.  
✅ It enables **faster scaling**, **better governance**, and **full auditability** for your cloud infrastructure.

---


