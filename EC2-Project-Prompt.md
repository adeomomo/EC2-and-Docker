# EC2 Project Prompt - Layer-by-Layer Build

You are helping me build an AWS EC2 project from scratch using Terraform, GitHub Actions, and AWS OIDC authentication. This project will eventually support Docker containers, but we're starting with a single EC2 instance and building layer by layer.

Each layer should result in a **fully working, testable system** that can be deployed and destroyed independently.

---

## Context and Learnings

I have completed a previous project building an S3 static website with:
- Terraform infrastructure as code
- GitHub Actions CI/CD
- AWS OIDC authentication (no hardcoded AWS keys)
- Remote Terraform state in S3 with DynamoDB locking
- Separate development and production strategies

I want to apply those same patterns to this EC2 project.

---

## Prerequisites Check

Before we begin building, help me verify and create (if missing) the following account-level resources:

### 1. GitHub OIDC Provider

Check if the GitHub OIDC provider exists:
```bash
aws iam list-open-id-connect-providers --profile admin
```

Expected output should contain:
```
arn:aws:iam::<account-id>:oidc-provider/token.actions.githubusercontent.com
```

**If missing:** Provide the command to create it.

### 2. Terraform State Bucket

Check if the state bucket exists:
```bash
aws s3 ls | grep terraform-state
```

Expected: A bucket named something like `<username>-terraform-state`

**If missing:** Provide commands to:
- Create the bucket
- Enable versioning
- Block public access
- Enable encryption

### 3. DynamoDB Lock Table

Check if the lock table exists:
```bash
aws dynamodb list-tables --profile admin
```

Expected: A table named something like `terraform-locks`

**If missing:** Provide the command to create it.

### 4. AWS CLI Profile Access

Verify the admin profile works:
```bash
aws sts get-caller-identity --profile admin
```

Expected: Shows the account ID and user ARN

**If any of these are missing, tell me exactly how to create them before proceeding.**

---

## Project Structure

This project will be built in layers:

```
Layer 1: IAM Role Setup
├── Create GitHub Actions IAM role
├── Create trust policy for this repo/branch
└── Grant basic EC2 permissions

Layer 2: Terraform Backend Configuration
├── Configure S3 backend
├── Test `terraform init` with remote state
└── Verify state is stored in S3

Layer 3: EC2 Instance Provisioning
├── Create security group
├── Create EC2 instance in default VPC
├── Output instance public IP
└── Test SSH access

Layer 4: Environment Separation
├── Create dev and prod state keys
├── Create separate IAM roles per environment
└── Deploy to both environments

Layer 5: Docker Preparation (Future)
├── IAM role for ECR access
├── Docker runtime on EC2
└── Pull and run containers
```

Each layer should be independently deployable and should not break previous layers.

---

## Layer 1: IAM Role and Trust Policy

### Objective
Create a GitHub Actions IAM role that this repository can assume, with minimal permissions to manage EC2 resources.

### Steps

1. **Create the IAM role** with a trust policy that allows only this GitHub repository on the main branch to assume it.

   - Role name: `GitHubActionsTerraformRole-ec2`
   - Trust the GitHub OIDC provider
   - Restrict to this repo and branch using the modern GitHub OIDC subject format (with numeric IDs)

2. **Create a permissions policy** that allows:
   - `ec2:*` for EC2 management
   - `ec2:DescribeSecurityGroups`
   - `ec2:DescribeInstances`
   - `ec2:DescribeVpcs`
   - S3 access for backend state

3. **Verify** the role and trust policy are correctly configured.

4. **Store the role ARN** as a GitHub secret: `AWS_ROLE_ARN`
5. **Hints**
    - Review previous policies
    - Use Amazon Q to help with access and trust policy syntax
    - Use least priv principles avoid Admin access
    - Copy the AWS ARN to github > repo > secrets > actions "AWS_ARN"

### Deliverable
- IAM role exists
- Trust policy is correct for this repo/branch
- GitHub secret `AWS_ROLE_ARN` is set
- `aws sts get-caller-identity` works with the role

---

## Layer 2: Terraform Backend and Project Setup

### Objective
Set up a working Terraform project with remote state, configured to deploy EC2 resources.

### Steps

1. **Create the repository structure:**
   ```
   ec2-project/
   ├── .github/
   │   └── workflows/
   │       └── terraform.yml
   ├── backend.tf
   ├── providers.tf
   ├── main.tf
   ├── variables.tf
   ├── outputs.tf
   ├── README.md
   └── .gitignore
   ```

2. **Configure backend.tf** to use the remote S3 state bucket with a unique state key for this project:
   ```hcl
   terraform {
     backend "s3" {
       bucket         = "<your-state-bucket>"
       key            = "ec2-project/main/terraform.tfstate"
       region         = "ap-southeast-2"
       encrypt        = true
       dynamodb_table = "terraform-locks"
     }
   }
   ```

3. **Configure providers.tf** (do NOT hardcode AWS profile):
   ```hcl
   terraform {
     required_version = ">= 1.0.0"
     required_providers {
       aws = {
         source  = "hashicorp/aws"
         version = "~> 5.0"
       }
     }
   }

   provider "aws" {
     region = "ap-southeast-2"
   }
   ```

4. **Create the GitHub Actions workflow** that:
   - Checks out code
   - Authenticates via OIDC
   - Runs `terraform init`, `terraform validate`, `terraform plan`
   - Applies on push to main

5. **Initialize Terraform locally:**
   ```bash
   export AWS_PROFILE=admin
   terraform init
   terraform validate
   terraform plan
   ```

   Expected: No resources yet, but Terraform successfully connects to the remote backend.

6. **Verify state is in S3:**
   ```bash
   aws s3 ls s3://<your-state-bucket>/ec2-project/main/
   ```

### Deliverable
- Repository created with correct structure
- `terraform init` works and uses remote S3 backend
- `terraform validate` passes
- `terraform plan` succeeds (no resources yet)
- GitHub Actions workflow is committed and ready
- State file exists in S3

---

## Layer 3: EC2 Instance Provisioning

### Objective
Create a minimal, working EC2 instance in the default VPC that can be deployed and destroyed.

### Steps

1. **Create variables.tf** with:
   - `instance_type` (default: `t2.micro`)
   - `ami_id` (default: latest Amazon Linux 2)
   - `instance_name` (default: `ec2-project-instance`)

2. **Create main.tf** with:
   - Security group allowing SSH (port 22) from your IP or 0.0.0.0/0 (for testing only)
   - Security group allowing outbound internet
   - EC2 instance in default VPC
   - Assign public IP

3. **Create outputs.tf** with:
   - Instance public IP
   - Instance ID
   - Security group ID

4. **Test locally:**
   ```bash
   terraform plan
   terraform apply
   ```

   Expected: EC2 instance is created with public IP.

5. **Verify in AWS Console:**
   - EC2 instance appears and is running
   - Security group is correct
   - Public IP is assigned

6. **Test SSH access:**
   ```bash
   ssh -i <your-key-pair> ec2-user@<public-ip>
   ```

   Note: You may need to create and download an EC2 key pair if you don't have one.

7. **Tear down:**
   ```bash
   terraform destroy
   ```

   Expected: EC2 instance is terminated.

8. **Push to main and test GitHub Actions:**
   - Commit all Terraform files
   - Push to main branch
   - Verify GitHub Actions workflow runs
   - Verify EC2 is created by GitHub Actions
   - Check AWS Console

### Deliverable
- EC2 instance can be created and destroyed via `terraform apply` and `terraform destroy`
- Public IP is output and SSH-accessible
- GitHub Actions successfully deploys and destroys
- Terraform state is maintained in S3

---

## Layer 4: Environment Separation (dev/prod)

### Objective
Support multiple environments (dev and prod) with separate state files and IAM roles.

### Steps

1. **Create separate state keys** in the backend configuration:
   - `ec2-project/dev/terraform.tfstate`
   - `ec2-project/prod/terraform.tfstate`

2. **Create separate GitHub Actions IAM roles:**
   - `GitHubActionsTerraformRole-ec2-dev`
   - `GitHubActionsTerraformRole-ec2-prod`

3. **Use GitHub environments** to separate deployments:
   - Environment: `development` → triggers on branch `develop`
   - Environment: `production` → triggers on branch `main` with approval

4. **Create separate Terraform workspaces or directories:**
   - Option A: Use `terraform workspace` (simple)
   - Option B: Create separate directories `./dev` and `./prod` (clearer)

5. **Test:**
   - Deploy to dev with `terraform apply`
   - Deploy to prod with `terraform apply`
   - Both environments have separate EC2 instances
   - Tear down dev without affecting prod

### Deliverable
- Separate dev and prod environments
- Separate state files for each
- Separate IAM roles for each
- Both environments work independently

---

## Checklist for Each Layer

Before moving to the next layer, verify:

- [ ] All Terraform files are committed to the repository
- [ ] `terraform init` succeeds
- [ ] `terraform validate` passes
- [ ] `terraform plan` shows expected changes
- [ ] `terraform apply` successfully creates resources
- [ ] Resources are visible in AWS Console
- [ ] GitHub Actions workflow runs successfully
- [ ] `terraform destroy` successfully removes all resources
- [ ] State file exists in the remote S3 backend
- [ ] No errors in CloudTrail or GitHub Actions logs

---

## Commands You'll Need

```bash
# Check prerequisites
aws sts get-caller-identity --profile admin
aws iam list-open-id-connect-providers --profile admin
aws s3 ls | grep terraform-state
aws dynamodb list-tables --profile admin

# Terraform workflow
terraform init -reconfigure
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
terraform destroy
terraform state list
terraform state show <resource>

# Verify in AWS
aws ec2 describe-instances --profile admin
aws ec2 describe-security-groups --profile admin

# Check GitHub Actions
# (View in GitHub Actions tab in your repository)

# Verify remote state
aws s3 ls s3://<your-state-bucket>/ec2-project/
```

---

## Debugging Tips

If something fails:

1. **Check AWS identity:**
   ```bash
   aws sts get-caller-identity --profile admin
   ```

2. **Check IAM role trust policy:**
   - Go to AWS Console → IAM → Roles → `GitHubActionsTerraformRole-ec2`
   - Verify the trust policy subject matches the GitHub OIDC token format

3. **Check CloudTrail for auth failures:**
   ```bash
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
     --max-results 10 \
     --profile admin
   ```

4. **Check Terraform state:**
   ```bash
   terraform state list
   terraform state show aws_instance.main
   ```

5. **Check GitHub Actions logs:**
   - Go to repository → Actions → Latest run → Click the job
   - Look for `terraform init`, `terraform plan`, or `terraform apply` output

---

## Success Criteria

When you complete all layers, you should have:

- ✅ A Terraform project that manages EC2 instances
- ✅ Remote state in S3 with locking
- ✅ GitHub Actions that deploys automatically
- ✅ OIDC authentication (no hardcoded AWS keys)
- ✅ Separate dev and prod environments
- ✅ Ability to create and destroy resources with `terraform apply/destroy`
- ✅ EC2 instance that is SSH-accessible
- ✅ Understanding of each layer and how they interact

---

## Next Steps After Layer 4

Once Layer 4 is working:

1. Add IAM role for EC2 to access ECR (prepare for Docker)
2. Add user data script to install Docker on EC2
3. Set up ECR repository
4. Push Docker image to ECR
5. Modify EC2 to pull and run Docker container

But for now, focus on getting Layer 1-4 solid.

---

## Final Notes

- **Build slowly**: Don't rush through layers. Test each one thoroughly.
- **Use the checklist**: Before moving to the next layer, verify everything works.
- **Ask for help**: If something fails, ask me to debug using CloudTrail, Terraform state, or AWS Console.
- **Document as you go**: Keep notes on what you learned at each layer.
- **Reuse patterns**: Every pattern here (OIDC, state, environments) will be reused in future AWS projects.

Good luck! Start with Layer 1 and let me know when you're ready for the next layer.
