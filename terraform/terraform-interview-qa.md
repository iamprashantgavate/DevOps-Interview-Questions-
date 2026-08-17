# Terraform Interview Q&A

## Q. How do you manage Terraform state in your project?

In my project, multiple DevOps engineers work on the same Terraform codebase. Storing the Terraform state file locally can cause conflicts and state corruption. Therefore, we use a remote backend with Amazon S3 and DynamoDB for Terraform state management.

The Terraform state file is stored in an Amazon S3 bucket, which provides centralized storage, versioning, and encryption. We enable S3 Versioning so that previous versions of the state file can be recovered if required.

```hcl
terraform {
  backend "s3" {
    bucket         = "healthcare-terraform-state-123"
    key            = "eks/prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

For state locking, we use a DynamoDB table. When someone runs `terraform apply`, Terraform creates a lock entry in the DynamoDB table to prevent multiple users from modifying the infrastructure simultaneously. This prevents state corruption.

We maintain separate state files for the Dev, Stage, and Production environments to ensure proper isolation.

If a state lock issue occurs, I first verify that no active deployment or Terraform execution is running. Only after proper validation do I use:

```bash
terraform force-unlock <LOCK_ID>
```

In our project, Terraform is executed through the GitHub Actions pipeline, not by individual engineers. This ensures that only one `terraform apply` operation runs at a time, reducing the possibility of state conflicts.

This setup provides secure, centralized, reliable, and collaborative Terraform state management for our AWS infrastructure.

---

## What are Terraform modules, and how do you structure them for multiple environments?

Terraform modules are reusable collections of Terraform configuration files that help reduce code duplication, improve maintainability, and standardize infrastructure provisioning.

In our project, the Terraform repository is organized with a `modules` directory and separate environment directories for Dev, Stage, and Production.

Instead of writing the same Terraform code repeatedly for resources such as VPC, EKS, IAM, Security Groups, S3, and RDS, we create reusable modules once and use them across multiple environments by passing different input variables.

Our project contains separate modules for:

- VPC
- EKS
- IAM
- Security Groups
- RDS

Each environment (Dev, Stage, and Production) has its own configuration that calls these modules with environment-specific values, allowing us to reuse the same code while keeping the environments isolated.

---

## Explain Terraform Import and when do you use it?

Terraform Import is used to bring an existing infrastructure resource that was created manually or outside Terraform into the Terraform state, so Terraform can start managing it.

We use Terraform Import when a resource already exists in AWS but is not currently tracked in the Terraform state.

Terraform Import does not create the resource. It only imports the existing resource into the Terraform state file.

**When do we use Terraform Import?**

- When a resource was created manually in AWS.
- When infrastructure already exists and we want Terraform to manage it.
- To avoid recreating production resources and prevent downtime.
- During migration from manually managed infrastructure to Infrastructure as Code (IaC).

**How does Terraform Import work?**

1. Write the Terraform configuration that matches the existing AWS resource.
2. Run the Terraform import command.
   ```bash
   terraform import aws_instance.web i-abcdef1234567890
   ```
3. Terraform imports the resource into the Terraform state file.
4. Run `terraform plan` to compare the Terraform configuration with the existing infrastructure.
5. Update the Terraform configuration until `terraform plan` shows `No changes`, ensuring the code matches the actual infrastructure.

This allows Terraform to manage the existing resource without recreating it.

---

## What is Terraform Drift? How do you detect drift and how do you fix it?

Terraform Drift occurs when the actual infrastructure in AWS differs from the infrastructure defined in the Terraform code and Terraform state. This usually happens when someone manually changes a resource through the AWS Console, AWS CLI, or another tool instead of using Terraform.

**How do you detect Terraform Drift?**

I detect Terraform Drift using the `terraform plan` command.

Terraform compares the Terraform configuration and state with the actual AWS infrastructure. If there are any differences, Terraform displays them in the plan output.

**How do you fix Terraform Drift?**

**Case 1: Manual changes are not required**

If the manual changes are not required, I simply run:

```bash
terraform apply
```

Terraform reverts the manual changes and brings the AWS infrastructure back in sync with the Terraform code and state.

**Case 2: Manual changes are required**

If the manual changes are valid and need to be retained:

- Update the Terraform code to match the current AWS infrastructure.
- Run `terraform plan` to verify that no further changes are required.
- Run `terraform apply` so the changes are recorded in the Terraform code and state.

**Case 3: Resource was created manually**

If a resource was created manually and is not managed by Terraform:

- Use `terraform import` to import the existing AWS resource into the Terraform state.
- Write the corresponding Terraform configuration for the resource.
- Run `terraform plan` and ensure it shows `No changes`.

This allows Terraform to manage the existing resource without recreating it.

---

## Explain a Terraform issue you handled in production

One Terraform issue I handled in production was related to Terraform state locking.

In our project, we use an Amazon S3 backend for storing the Terraform state file and DynamoDB for state locking.

During a production infrastructure deployment, the GitHub Actions pipeline failed with the following error:

> "Error acquiring the state lock."

My first step was to verify whether another Terraform deployment was already running because Terraform creates a state lock in DynamoDB to prevent concurrent modifications.

After checking the GitHub Actions pipeline and confirming that no deployment was in progress, I investigated the DynamoDB lock table and found a stale lock entry left behind by a previous failed deployment.

After confirming with the team that no Terraform execution was running, I removed the lock using:

```bash
terraform force-unlock <LOCK_ID>
```

I then re-ran the GitHub Actions pipeline, and the deployment completed successfully.

This incident reinforced the importance of verifying that no active Terraform execution is running before using `terraform force-unlock`, as removing an active lock can lead to state corruption.

**Another Terraform issue I handled**

I also faced a Terraform Drift issue where manual changes made in AWS caused a mismatch between the actual infrastructure and the Terraform configuration.

I detected the drift using:

```bash
terraform plan
```

After verifying the changes, I updated the Terraform configuration to match the actual infrastructure and ran `terraform apply` to bring the Terraform code, state, and AWS infrastructure back into sync without impacting production.

---

## How is Terraform used in your current project?

In my current project, we use Terraform as our Infrastructure as Code (IaC) tool to provision and manage AWS infrastructure. Instead of creating resources manually through the AWS Console, all infrastructure is defined in Terraform code and stored in a Git repository.

We follow a modular approach, where reusable modules are created for common components such as VPCs, EKS clusters, IAM roles, security groups, RDS, and S3 buckets. This helps maintain consistency across different environments like development, staging, and production.

Our Terraform state is stored remotely in an S3 bucket, and we use a DynamoDB table for state locking. This prevents multiple team members from modifying the same infrastructure simultaneously.

Whenever infrastructure changes are required, a developer or DevOps engineer creates a feature branch, makes the Terraform code changes, and raises a Pull Request. The changes are reviewed by another team member before being merged.

Once the PR is approved and merged into the main branch, our CI/CD pipeline automatically runs `terraform fmt`, `terraform validate`, `terraform plan`, and after approval, `terraform apply`. This ensures infrastructure changes are consistent, auditable, and version-controlled.

We also use environment-specific variable files (`dev.tfvars`, `stage.tfvars`, `prod.tfvars`) and store sensitive information such as passwords and API keys in AWS Secrets Manager or Parameter Store instead of hardcoding them in Terraform.

In addition, we regularly run `terraform plan` to detect configuration drift and ensure that the actual infrastructure matches the Terraform state.

---

## Q. How does your team use Terraform in a large environment?

In our organization, Terraform is integrated with our CI/CD pipeline, and all infrastructure changes are managed through Git. We follow GitOps principles, meaning no one makes infrastructure changes directly from their local machine or the AWS Console.

We organize our Terraform code into reusable modules for components like VPC, EKS, IAM, RDS, Security Groups, and Load Balancers. Different environments such as development, staging, and production have separate configurations and state files.

When a change is needed, a DevOps engineer creates a feature branch, updates the Terraform code, and raises a Pull Request. The CI/CD pipeline automatically performs:

- `terraform fmt` to ensure consistent formatting.
- `terraform validate` to check syntax and configuration.
- Static analysis and security scanning (for example, Checkov or TFLint).
- `terraform plan` to generate an execution plan.

The generated plan is attached to the Pull Request so reviewers can clearly see what infrastructure will be created, modified, or destroyed.

After the Pull Request is approved and merged, the deployment pipeline runs `terraform apply`. For production environments, we usually have a manual approval stage before applying changes.

Terraform state is stored remotely in an S3 bucket with DynamoDB used for state locking. This prevents concurrent modifications and keeps the state secure.

We also use environment-specific variables and workspaces or separate backend configurations, depending on the project. Sensitive information such as database passwords and API keys is stored in AWS Secrets Manager or Systems Manager Parameter Store instead of being hardcoded in Terraform.

This workflow provides version control, peer review, auditability, security, and consistent infrastructure deployments across all environments.
