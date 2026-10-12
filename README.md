# Terraform All Concepts — Complete Guide for Interviews and Practical Coding

For your DevOps Engineer interview, focus on understanding Terraform concepts with simple explanations, syntax, and real AWS examples. You don't need to memorize everything; learn when and why to use each concept.

I'll use an example of creating AWS EC2 instances, VPCs, subnets, and security groups.

## 1. Terraform concepts you must know

Core Terraform concepts

`count` and `for_each`

Create multiple resources without repeating code.

Variables, locals and outputs

Pass values, reuse expressions and display results.

Resources, data sources and dependencies

Create infrastructure, retrieve existing resources and control creation order.

Modules, state and backends

Reuse code, track infrastructure and store state safely.

Lifecycle, provisioners and dynamic blocks

Control resource behavior and generate configuration dynamically.

## 2. `count` — Create multiple resources

Meaning: `count` creates a specified number of copies of a resource.

Example: Create three EC2 instances.

```
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"
  }
}
```

Here, `count = 3` creates three instances. `count.index` starts from `0`.

| Instance | Name    |
| -------- | ------- |
| First    | `web-0` |
| Second   | `web-1` |
| Third    | `web-2` |

When to use: When you need multiple similar resources and a numeric index is sufficient.

Interview point: If you remove an item from a list used with `count`, Terraform may change or replace resources associated with later indexes.

## 3. `for_each` — Create resources using keys

Meaning: `for_each` creates multiple resources using a map or a set of strings.

Example: Create three EC2 instances with different names.

```
resource "aws_instance" "web" {
  for_each = toset(["dev", "stage", "prod"])

  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = each.key
  }
}
```

This creates instances named `dev`, `stage`, and `prod`.

Important expressions:

- `each.key` — the current key.
- `each.value` — the current value.

For a map, you can use different configurations for different instances:

```
resource "aws_instance" "web" {
  for_each = {
    app1 = {
      subnet_id = "subnet-111"
      type      = "t3.micro"
    }
    app2 = {
      subnet_id = "subnet-222"
      type      = "t3.small"
    }
  }

  ami           = var.ami_id
  instance_type = each.value.type
  subnet_id     = each.value.subnet_id

  tags = {
    Name = each.key
  }
}
```

Here, each instance can have its own subnet and instance type.

## 4. `count` vs `for_each`

| `count`                      | `for_each`                                       |
| ---------------------------- | ------------------------------------------------ |
| Uses a number                | Uses a map or set of strings                     |
| Access using `count.index`   | Access using `each.key` and `each.value`         |
| Good for identical resources | Good for named resources with different settings |
| Uses numeric indexes         | Uses stable keys when the keys remain unchanged  |

Interview answer: “I use `count` when I need a fixed number of similar resources. I use `for_each` when resources have different names or configurations. For named infrastructure, I generally prefer `for_each` because its keys make resource identity easier to manage.”

## 5. Variables — `variable`

Meaning: Variables let you pass values into Terraform instead of hardcoding them.

```
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

Here, `var.instance_type` reads the variable value.

Common variable types:

```
variable "name" {
  type = string
}

variable "instance_count" {
  type = number
}

variable "enabled" {
  type = bool
}

variable "subnet_ids" {
  type = list(string)
}

variable "tags" {
  type = map(string)
}
```

Pass values through `terraform.tfvars`:

```
instance_type = "t3.small"
instance_count = 3
enabled = true
subnet_ids = ["subnet-111", "subnet-222"]
tags = {
  Environment = "dev"
}
```

Real example: Use the same Terraform code for Dev and Prod, but pass different instance types and subnet IDs.

## 6. `locals` — Reusable values

Meaning: Locals let you define an expression once and reuse it.

```
locals {
  project_name = "flask-app"
  environment  = "dev"

  common_tags = {
    Project     = local.project_name
    Environment = local.environment
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  tags          = local.common_tags
}
```

- Variables receive input.
- Locals calculate or organize reusable values.
- Use `var.name` for variables and `local.name` for locals.

## 7. `output` — Display resource information

Meaning: Outputs display useful information after Terraform creates resources.

```
output "instance_id" {
  value = aws_instance.web.id
}

output "public_ip" {
  value = aws_instance.web.public_ip
}
```

After applying, you can retrieve an output with:

```
terraform output
terraform output public_ip
```

Real example: Display an EC2 public IP so you know where to access your application.

## 8. Resource vs Data Source

A resource creates or manages infrastructure. A data source reads information about existing infrastructure.

Resource example:

```
resource "aws_s3_bucket" "app" {
  bucket = "my-unique-app-bucket-example"
}
```

Terraform manages this bucket.

Data source example:

```
data "aws_vpc" "existing" {
  id = var.existing_vpc_id
}
```

Terraform reads information about the existing VPC; this block does not create it.

You can then use it:

```
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  subnet_id     = var.existing_subnet_id

  # Other required settings can go here.
}
```

When to use a data source: When you want Terraform to use an existing AMI, VPC, subnet, or another supported resource without creating it through that data block.

## 9. Dependencies — `depends_on`

Meaning: Tells Terraform that one resource must be handled after another dependency.

```
resource "aws_s3_bucket" "app" {
  bucket = "my-unique-app-bucket-example"
}

resource "aws_s3_bucket_public_access_block" "app" {
  bucket = aws_s3_bucket.app.id

  depends_on = [aws_s3_bucket.app]
}
```

In this example, the bucket reference already creates an implicit dependency. Therefore, the explicit `depends_on` is redundant; it illustrates the syntax.

There are two types:

- Implicit dependency: Terraform detects a reference such as `aws_s3_bucket.app.id`.
- Explicit dependency: You declare `depends_on` when a dependency is not expressed through a direct reference.

Interview tip: Prefer implicit dependencies where possible. Use explicit dependencies when Terraform cannot infer the required ordering.

## 10. Conditional expressions — `condition ? true : false`

Meaning: Select one value based on a condition.

```
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  instance_type = var.environment == "prod" ? "t3.medium" : "t3.micro"
}
```

If the environment is `prod`, the type is `t3.medium`; otherwise, it is `t3.micro`.

The format is:

```
condition ? value_if_true : value_if_false
```

Real example: Use larger EC2 instances in production and smaller instances in development.

## 11. `dynamic` blocks — Generate nested blocks

Meaning: A dynamic block generates repeatable nested configuration blocks from a collection.

For example, a security group may need multiple ingress rules.

```
variable "ingress_rules" {
  type = list(number)
  default = [80, 443]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
    }
  }
}
```

This generates ingress blocks for ports 80 and 443.

Important difference:

- `for_each` on a resource creates multiple resources.
- `dynamic` generates nested blocks inside a resource.

Use dynamic blocks only when they genuinely reduce repetitive nested configuration.

## 12. `lifecycle` — Control resource behavior

Meaning: Lifecycle rules tell Terraform how to handle changes to a resource.

### A. `prevent_destroy`

Prevents Terraform from destroying a resource while the rule remains in its configuration.

```
resource "aws_s3_bucket" "important" {
  bucket = "my-unique-important-bucket"

  lifecycle {
    prevent_destroy = true
  }
}
```

Useful for protecting important resources from accidental destruction.

### B. `create_before_destroy`

Creates a replacement before destroying the old resource when Terraform must replace it and the provider/API allows this ordering.

```
lifecycle {
  create_before_destroy = true
}
```

Useful when you want to reduce downtime during replacement.

### C. `ignore_changes`

Ignores changes to selected attributes during normal update planning.

```
lifecycle {
  ignore_changes = [tags]
}
```

Useful when another approved process manages a particular attribute.

Warning: These rules do not replace backups, access controls, review, or disaster recovery. `prevent_destroy` can also block legitimate changes.

## 13. Modules — Reuse Terraform code

Meaning: A module is a collection of Terraform configuration files that work together.

Example folder structure:

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── modules/
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Call the module:

```
module "app_server" {
  source        = "./modules/ec2"
  ami_id        = var.ami_id
  instance_type = "t3.micro"
}
```

Inside the module, declare the inputs:

```
# modules/ec2/variables.tf

variable "ami_id" {
  type = string
}

variable "instance_type" {
  type = string
}
```

And create the resource in `modules/ec2/main.tf`:

```
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```

Real example: Create one reusable EC2 module and call it for Dev, Staging, and Production.

Remember: a child module has its own inputs and outputs, and its outputs must be declared before a caller can access them.

## 14. Terraform state — `terraform.tfstate`

Meaning: State records Terraform's understanding of the resources it manages and their relationship to your configuration.

Common state-related concepts:

- `terraform.tfstate` — local state file, when using local state.
- `terraform.tfstate.backup` — a backup file that may be created with local state operations.
- Remote state — state stored in a backend, such as an S3 bucket.
- State locking — helps prevent concurrent state-writing operations when supported by the backend.

Interview question: What happens if you delete the state file?

Terraform may lose its tracking information and can propose creating resources that already exist. Actual behavior depends on configuration, provider refresh, and backend setup. Never delete state casually.

Useful commands:

```
terraform state list
terraform state show aws_instance.web
terraform state pull
```

Treat state as sensitive because it can contain passwords, tokens, or other sensitive resource attributes.

## 15. Backend — Store state remotely

Meaning: A backend determines where Terraform stores state and how state operations are handled.

Example: S3 backend.

```
terraform {
  backend "s3" {
    bucket       = "my-terraform-state-bucket"
    key          = "dev/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
  }
}
```

This is a modern S3 backend configuration using S3 lockfiles. The bucket must already exist, and the AWS credentials must have the required permissions.

After configuring or changing a backend, run:

```
terraform init
```

Why remote state?

- Teams can share the same state.
- State can be centrally managed.
- Supported backends can provide locking.
- State access can be controlled through IAM and backend permissions.

Do not commit state files or credentials to Git.

## 16. `terraform.workspace` — Workspace-specific configuration

Meaning: A workspace gives Terraform a separate state instance for a configuration.

```
locals {
  environment = terraform.workspace
}
```

Common commands:

```
terraform workspace list
terraform workspace new dev
terraform workspace select dev
```

Workspaces can be useful for similar deployments with separate state. However, they do not automatically isolate credentials, permissions, backend configuration, or every environment-specific setting.

For strongly isolated production environments, separate configurations and appropriately separated backends or accounts may be more suitable.

## 17. `for` expressions — Transform lists and maps

Meaning: A `for` expression loops over values and creates a new list or map.

```
variable "names" {
  default = ["app1", "app2", "app3"]
}

output "upper_names" {
  value = [for name in var.names : upper(name)]
}
```

Output:

```
["APP1", "APP2", "APP3"]
```

You can also create a map:

```
locals {
  instance_map = {
    for name in var.names : name => "t3.micro"
  }
}
```

Difference: `for_each` creates resources or instances of a module; a `for` expression transforms data.

## 18. `toset`, `tolist`, `tomap` and `tomember`

Terraform has conversion functions for working with collections. A common one is `toset()`.

```
for_each = toset(["dev", "stage", "prod"])
```

- `toset()` — converts a collection to a set of unique values.
- `tolist()` — converts a compatible collection to a list.
- `tomap()` — converts a compatible value to a map.
- `tomember()` is not a standard Terraform conversion function.

A set removes duplicates and does not preserve list ordering.

## 19. `templatefile()` — Generate files from templates

Meaning: Reads a template file and replaces its placeholders with supplied values.

```
locals {
  user_data = templatefile("${path.module}/user_data.tftpl", {
    app_name = "flask"
  })
}
```

The template file `user_data.tftpl` might contain:

```
#!/bin/bash
echo "Installing ${app_name}"
```

You can use `local.user_data` in an EC2 resource's `user_data` argument.

Use case: Generate startup scripts or configuration files using environment-specific values.

## 20. `file()` and `fileexists()`

```
locals {
  script = file("${path.module}/install.sh")
}
```

- `file()` reads an existing file as a string.
- `fileexists()` checks whether a file exists.

These functions read files available when Terraform evaluates the configuration. They are not intended to read files generated later during the same apply operation.

## 21. `try()` and `can()` — Handle expressions safely

### `try()`

Returns the first expression that does not produce a dynamic evaluation error.

```
locals {
  app_name = try(var.config.app_name, "default-app")
}
```

If the first expression cannot be evaluated, Terraform can use `"default-app"`.

### `can()`

Returns `true` if an expression can be evaluated without an error; otherwise, it returns `false`.

```
variable "port" {
  type = number

  validation {
    condition     = can(var.port >= 1 && var.port <= 65535)
    error_message = "Port must be between 1 and 65535."
  }
}
```

This example uses a variable validation block to check the port range. Because the variable is already typed as a number, the `can()` wrapper is unnecessary here; the range check alone is sufficient.

## 22. Variable validation and sensitive values

You can validate inputs and mark sensitive values to reduce accidental disclosure.

```
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "stage", "prod"], var.environment)
    error_message = "Use dev, stage, or prod."
  }
}
```

Sensitive value:

```
variable "db_password" {
  type      = string
  sensitive = true
}
```

Important: `sensitive = true` redacts values in many Terraform outputs, but it does not encrypt the value automatically or prevent it from being stored in state.

Use an appropriate secrets manager and protect state access.

## 23. Provisioners — `remote-exec` and `local-exec`

Provisioners run commands during certain resource lifecycle events.

Local example:

```
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo EC2 created"
  }
}
```

This executes a command on the machine running Terraform when the provisioner runs.

- `local-exec` — executes on the Terraform runner.
- `remote-exec` — executes commands on a remote machine, typically over SSH or WinRM.

Interview answer: Provisioners are generally a last resort. Prefer cloud-init/user data, configuration management tools, images, or other purpose-built automation where appropriate.

## 24. `terraform import` — Import existing infrastructure

Meaning: Import associates an existing resource with a Terraform resource address.

Example:

```
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}
```

You must also declare a matching resource block:

```
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}
```

Then run:

```
terraform plan
terraform apply
```

Review the plan carefully. Importing does not automatically generate a complete configuration or guarantee that the configuration matches every existing setting.

You can also use the CLI:

```
terraform import aws_s3_bucket.existing my-existing-bucket
```

## 25. `moved` block — Rename a resource safely

Suppose you rename a Terraform resource from `web` to `app`.

```
moved {
  from = aws_instance.web
  to   = aws_instance.app
}
```

This tells Terraform that the existing resource address has changed. It helps avoid treating a simple configuration rename as a request to destroy and recreate the resource.

## 26. `replace_triggered_by` and `replace`

Terraform also supports more advanced resource replacement controls.

```
lifecycle {
  replace_triggered_by = [
    aws_security_group.web
  ]
}
```

This tells Terraform to replace the resource containing this lifecycle rule when the referenced resource has a planned change that triggers replacement.

For a one-time replacement, you can use:

```
terraform plan -replace="aws_instance.web"
```

Review the plan before applying. A forced replacement can cause downtime or data loss depending on the resource.

## 27. `dynamic` vs `for_each` vs `count` vs `for`

This is a very common interview comparison.

| Feature          | Main purpose                                  |
| ---------------- | --------------------------------------------- |
| `count`          | Create a numbered set of resource instances   |
| `for_each`       | Create resource instances using keys          |
| `for` expression | Transform collection values                   |
| `dynamic`        | Generate repeated nested configuration blocks |

Example to remember:

- Ten EC2 instances → `count` or `for_each`.
- Three named environments → often `for_each`.
- Convert names to uppercase → `for`.
- Generate multiple security group ingress blocks → `dynamic`.

## 28. Other important Terraform concepts

| Concept              | Simple meaning                                                            |
| -------------------- | ------------------------------------------------------------------------- |
| `provider`           | Connects Terraform to AWS, Azure, GCP, or another platform                |
| `required_providers` | Declares provider source and version constraints                          |
| `required_version`   | Restricts which Terraform CLI versions are supported                      |
| `terraform.tfvars`   | Supplies variable values                                                  |
| `*.auto.tfvars`      | Automatically loaded variable values                                      |
| `terraform.tfstate`  | Local state file when using the local backend                             |
| `terraform.lock.hcl` | Records selected provider versions and checksums                          |
| `depends_on`         | Specifies an explicit dependency                                          |
| `locals`             | Stores reusable expressions                                               |
| `output`             | Exposes useful results                                                    |
| `lifecycle`          | Controls resource lifecycle behavior                                      |
| `check` blocks       | Define infrastructure assertions that can report failed checks            |
| `import` blocks      | Declare imports of existing infrastructure                                |
| `removed` blocks     | Manage removal of resources from state, with optional destruction         |
| `terraform_data`     | Provides a built-in resource for lifecycle behavior and related workflows |

### Provider aliases

Use aliases when you need multiple AWS provider configurations, such as different regions.

```
provider "aws" {
  region = "ap-south-1"
}

provider "aws" {
  alias  = "mumbai_backup"
  region = "ap-southeast-1"
}

resource "aws_s3_bucket" "backup" {
  provider = aws.mumbai_backup
  bucket   = "my-unique-backup-bucket-example"
}
```

The second provider configuration is explicitly selected for the backup bucket.

### `merge()` — Combine maps

```
locals {
  default_tags = {
    Project = "flask"
  }

  environment_tags = {
    Environment = "dev"
  }

  tags = merge(local.default_tags, local.environment_tags)
}
```

This combines the maps into a single map.

### `lookup()` — Retrieve a map value

```
locals {
  instance_types = {
    dev  = "t3.micro"
    prod = "t3.medium"
  }

  selected_type = lookup(local.instance_types, var.environment, "t3.micro")
}
```

The third argument is the default value if the key isn't present.

### `length()` and `contains()`

```
length(["app1", "app2", "app3"])
```

Returns `3`.

```
contains(["dev", "stage", "prod"], "prod")
```

Returns `true`.

Other useful functions include `join()`, `split()`, `flatten()`, `concat()`, `keys()`, `values()`, `format()`, `upper()`, `lower()`, `replace()`, `distinct()` and `zipmap()`.

## 29. Terraform file structure

For a practical AWS project, you can organize files like this:

```
terraform-project/
├── versions.tf
├── provider.tf
├── backend.tf
├── variables.tf
├── terraform.tfvars
├── locals.tf
├── vpc.tf
├── security_groups.tf
├── ec2.tf
├── outputs.tf
└── modules/
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Terraform automatically loads `.tf` files in the same directory. These filenames are organizational conventions; Terraform does not require every resource type to have a separate file.

## 30. Terraform commands — Learn the workflow

1\. `terraform init`

Initializes the working directory, downloads providers and initializes the backend.

```
terraform init
```

2\. `terraform fmt` and `validate`

Formats the configuration and checks its internal consistency.

```
terraform fmt
terraform validate
```

3\. `terraform plan`

Shows the proposed infrastructure changes.

```
terraform plan
terraform plan -out=tfplan
```

4\. `terraform apply`

Executes the planned changes after approval, when required.

```
terraform apply
terraform apply tfplan
```

5\. Inspect and clean up

Inspect outputs and state. Destroy only when you intentionally want to remove managed infrastructure.

```
terraform output
terraform state list
terraform destroy
```

## 31. Quick practice test

Try answering these without looking at the guide.

1\. You need 10 identical EC2 instances. Which feature can you use?

output

count

locals

2\. You need three named instances with different subnet IDs. Which is generally better?

for_each

terraform output

terraform fmt

3\. You want to read an existing VPC without creating one. What do you use?

resource

data source

provisioner

4\. You need to generate multiple ingress blocks inside one security group. What do you use?

dynamic block

count.index

output

5\. Which file normally tracks managed resources when using local state?

terraform.lock.hcl

terraform.tfvars

terraform.tfstate

6\. What should you normally run to review changes before deployment?

terraform destroy

terraform plan

terraform output

Try again

My recommendation: For your coding test, prioritize `count`, `for_each`, variables, outputs, locals, data sources, dependencies, dynamic blocks, lifecycle, modules, state, and the `init → validate → plan → apply` workflow. Practise writing these from memory in an actual AWS Terraform project.
