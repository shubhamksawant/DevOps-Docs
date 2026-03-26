## Terraform Advanced Scenario-Based Questions & Answers (2025)



### 1. Two developers run `terraform apply` on the same module at the same time. What happens and how do you prevent state corruption?

**Behavior**

- **First apply**: Acquires the state lock and proceeds.
- **Second apply**: Fails with a lock error, for example:

```text
Error: Error acquiring the state lock
```

**How to prevent corruption**

- **Use a remote backend with state locking**:
  - **AWS S3 + DynamoDB** (DynamoDB table for state locking)
  - **Terraform Cloud / Enterprise**
  - **Consul** (as a backend with locking)
- **Best practice**: Run Terraform via a **CI/CD pipeline** and **block local applies** to ensure serialized, auditable changes.

---

### 2. Changing the AMI ID in a Launch Template forces ASG recreation. How do you achieve zero (or near-zero) downtime?

**Goal**: Update the Auto Scaling Group (ASG) to use a new AMI while avoiding downtime.

**Solution**

- Use `create_before_destroy` to create new resources before destroying the old ones:

```hcl
lifecycle {
  create_before_destroy = true
}
```

- Optionally ignore Launch Template version drift:

```hcl
lifecycle {
  ignore_changes = [latest_version]
}
```

**Why**

- ASG updates to its Launch Template often trigger **replacement**.
- `create_before_destroy` ensures new instances are launched and become healthy **before** old instances are terminated.

---

### 3. Terraform wants to recreate an entire RDS instance due to a minor parameter change, but you must avoid downtime. What do you do?

**Problem**: Some RDS attributes force replacement; you only want a non-disruptive parameter change.

**Solution**

- Ignore specific parameter changes at the DB instance level:

```hcl
lifecycle {
  ignore_changes = [parameter_group_name]
}
```

- Or move configuration into a separate **DB parameter group** and update that instead of the instance resource.

---

### 4. After `terraform import` of an EC2 instance, every `plan` shows changes even though AWS hasn’t changed. Why?

**Reason**

- `terraform import` synchronizes **state only**, not your **code**.
- Your resource block does not exactly match the real resource attributes.

**Fix**

1. Run:

   ```bash
   terraform show
   ```

2. Inspect the imported resource’s attributes.
3. Update the Terraform resource block so that all relevant arguments match the **actual** values.
4. Run `terraform plan` until the plan is clean (no changes).

---

### 5. Terraform marks a resource as tainted and keeps recreating it on every apply. How do you find and fix the cause?

**How to inspect**

```bash
terraform state show <resource_address>
```

**Common reasons**

- **Drift**: The resource was changed manually in the cloud.
- **Missing attributes** in the code that Terraform infers as changed.
- **Provider bug** or flapping computed values.
- A field is **computed** and changes on every refresh.

**Fix**

- Explicitly replace the resource once:

```bash
terraform apply -replace="aws_instance.example"
```

- Then adjust your code (or `ignore_changes`) to prevent repeated “differences”.

---

### 6. One module deploys VPCs for dev, stage, and prod. Dev/stage use 2 subnets, prod uses 6. How do you design the module?

**Goal**: Reuse a single module with a different number of subnets per environment.

**Solution**

- Use `for_each` with a map and avoid `count` (which is order-sensitive and can cause recreation):

```hcl
variable "subnets" {
  type = map(list(string))
}

resource "aws_subnet" "this" {
  for_each = toset(var.subnets[var.env])

  vpc_id     = var.vpc_id
  cidr_block = each.value
  # ... other arguments ...
}
```
It makes total sense why you'd want to steer clear of count here. While count is the "quick and dirty" way to scale resources, it treats your subnets like an array. If you remove the first item in that array, Terraform panics and tries to shift or recreate everything that follows. Using for_each with a map makes your infrastructure "key-addressable," which is much safer.

Here is how you can structure the code to handle your dev, stage, and prod requirements dynamically.

1. The Variable Structure
Instead of a simple list, we use a map where the keys are your environment names. This allows the module to look up exactly what it needs based on a single input (like var.env).

Terraform
variable "env" {
  description = "The environment name (dev, stage, or prod)"
  type        = string
}

variable "env_config" {
  type = map(object({
    cidr_block = string
    subnets    = map(string) # Map of subnet_name => cidr
  }))
  default = {
    dev = {
      cidr_block = "10.0.0.0/16"
      subnets    = { "dev-app-1" = "10.0.1.0/24", "dev-app-2" = "10.0.2.0/24" }
    }
    prod = {
      cidr_block = "10.1.0.0/16"
      subnets    = {
        "p-app-1" = "10.1.1.0/24", "p-app-2" = "10.1.2.0/24",
        "p-db-1"  = "10.1.3.0/24",  "p-db-2"  = "10.1.4.0/24",
        "p-web-1" = "10.1.5.0/24", "p-web-2" = "10.1.6.0/24"
      }
    }
  }
}
2. Resource Logic with for_each
This block will look at the env_config map, find the current environment, and loop through the subnets map defined for it.

Terraform
resource "aws_vpc" "main" {
  cidr_block = var.env_config[var.env].cidr_block
}

resource "aws_subnet" "this" {
  # We iterate over the subnets map for the specific environment
  for_each = var.env_config[var.env].subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = "us-east-1a" # You could also map these!

  tags = {
    Name = each.key
  }
}
3. Using dynamic Blocks
Dynamic blocks are useful when a resource has nested configuration blocks (like ingress rules in a Security Group) that need to change based on the environment.

Terraform
resource "aws_security_group" "allow_tls" {
  name   = "${var.env}-sg"
  vpc_id = aws_vpc.main.id

  # Generate multiple ingress blocks based on the environment
  dynamic "ingress" {
    for_each = var.env == "prod" ? [443, 80] : [443, 80, 8080, 22]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
Why count is risky here
Imagine you used count = 6 for your Prod subnets. Terraform tracks them by index:

aws_subnet.this[0]

aws_subnet.this[1]

...and so on.

The Issue: If you decide to delete the first subnet (index 0), Terraform doesn't just delete one. It sees that the list has shifted. It will try to rename index 1 to index 0, index 2 to index 1, etc. In many cases, this forces Terraform to destroy and recreate every single subnet in the list just because the order changed.

The for_each Advantage: Terraform tracks resources by the key (e.g., aws_subnet.this["p-db-1"]). If you delete one subnet, Terraform sees that specific key is gone, deletes only that one, and leaves the others completely untouched.

Would you like me to show you how to dynamically distribute these subnets across multiple Availability Zones using the cidrsubnets() function?
- Optionally use **dynamic blocks** for nested arguments that vary per environment.

---

### 7. You want to deploy the same infra across 10 AWS accounts from one code base. How do you design it?

**Solution**

- Define multiple AWS providers with aliases:

```hcl
provider "aws" {
  alias  = "acc1"
  region = "ap-south-1"
}

provider "aws" {
  alias  = "acc2"
  region = "eu-west-1"
}
```

- Override the provider in module calls:

```hcl
module "network_acc1" {
  source    = "./network"
  providers = { aws = aws.acc1 }
  # ...variables...
}
```

- Use **STS AssumeRole** in provider configuration for secure, cross-account access.

---

### 8. Terraform plan is slow because it queries thousands of resources. How do you optimize performance?

**Strategies**

- Use `lifecycle ignore_changes` for attributes that don’t matter for your drift detection.
- Use **data sources only when required**; avoid overusing them to fetch large inventories.
- **Split** the configuration into multiple root modules / state files:
  - Separate state for “network”, “database”, “app”, etc.
  - Use **remote state data sources** to share outputs.
  - Optionally use **Terragrunt** to orchestrate multiple stacks.

---

### 9. A sensitive variable (like a password) accidentally got committed to Git. How do you prevent this in the future?

**Terraform configuration**

```hcl
variable "password" {
  type      = string
  sensitive = true
}
```

**Practices to prevent future leaks**

- Add `terraform.tfvars` (and any secrets files) to **`.gitignore`**.
- Use **pre-commit hooks** to block committing secrets.
- Store secrets in:
  - **HashiCorp Vault**
  - **AWS SSM Parameter Store**
  - **AWS Secrets Manager** or similar secret managers

---

### 10. Terraform wants to recreate an ALB every time because AWS injects default tags. How do you stop this?

**Problem**

- AWS adds or modifies **default tags**, which causes perpetual diffs.

**Solution**

- Ignore provider- or AWS-managed tags:

```hcl
lifecycle {
  ignore_changes = [tags]
}
```

**Note**: This pattern is common for **EKS**, **ALB**, **EC2**, **S3**, and other managed resources that auto-add tags.

---

### 11. You want to share outputs from one Terraform project into another. How do you do it?

**Solution: Remote state data source**

```hcl
data "terraform_remote_state" "vpc" {
  backend = "s3"

  config = {
    bucket = "tf-state"
    key    = "vpc/terraform.tfstate"
    region = "ap-south-1"
  }
}
```

Then you can consume:

```hcl
vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id
```

---

### 12. `terraform destroy` is stuck because RDS requires a final snapshot. How do you handle this?

**Options**

- Skip final snapshot:

```hcl
skip_final_snapshot = true
```

- Or explicitly create a final snapshot:

```hcl
final_snapshot_identifier = "backup-before-destroy"
```

---

### 13. Terraform keeps recreating an S3 bucket because AWS changes ACL/ownership defaults. How do you stabilize it?

**Problem**

- AWS’s S3 **Object Ownership** and ACL features can auto-adjust settings, causing drift.

**Solution**

- Ignore those fields:

```hcl
lifecycle {
  ignore_changes = [
    acl,
    grant,
    object_ownership
  ]
}
```

This is very common after AWS introduced S3 Object Ownership defaults.

---

### 14. Terraform cannot delete a resource because it’s still attached (e.g., ENI, IAM role). How do you resolve cyclic dependencies?

**Approaches**

- Use explicit `depends_on` to control order:

```hcl
depends_on = [
  aws_iam_role_policy.this,
  aws_security_group.this
]
```

- If necessary, **split** the teardown into multiple phases:
  - First detach or delete dependent resources.
  - Then destroy the primary resource.

---

### 15. You have 10 environment folders (dev → prod) with high code duplication. How do you make it DRY?

**Solution: Use Terragrunt**

- Example layout:

```text
/live/dev/network/terragrunt.hcl
/live/prod/network/terragrunt.hcl
/modules/network/main.tf
```

- Terragrunt helps with:
  - **Remote state** configuration
  - **State locking**
  - **Backend config** reuse
  - **Environment variables**
  - A DRY, hierarchical structure for multiple environments


