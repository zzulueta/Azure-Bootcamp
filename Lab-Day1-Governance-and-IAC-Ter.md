---
lab:
  title: 'Lab Day 1: Governance & Infrastructure as Code (Terraform)'
  module: 'Advanced Azure Bootcamp – Day 1'
---

# Lab Day 1 – Governance & Infrastructure as Code (Terraform)

## Lab Introduction

In this lab you build a governance foundation in Azure and then deploy infrastructure
using Terraform as the primary Infrastructure as Code language. You will work through the following tasks:
- Configure RBAC with least-privilege role assignment
- Enforce standards with Azure Policy, tags, and resource locks
- Understand Terraform language constructs: providers, variables, locals, resources, data sources, and outputs
- Author a Terraform configuration from scratch using variable validation, conditions, and string interpolation
- Deploy Terraform configurations via Cloud Shell using environment-specific variable files and workspaces
- Apply deployment best practices — `terraform plan`, `.tfvars` files, formatting, validation, and state

## Pre-requisites
- An Azure subscription
- A resource group named **RG-Lab1.** Note: You will be assigned your own unique resource group name in the sandbox environment to avoid conflicts with other participants. Create a resource group in the same region you will deploy resources to (e.g., Australia East). Name the resource group **RG-Lab1-yourname**. Note the resource group name for use in all lab tasks.
- Owner rights at the resource group level
- Terraform CLI v1.5 or later (pre-installed in Azure Cloud Shell)

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is formalising its Azure landing zone ahead of a production workload migration. Before any infrastructure can be deployed, governance guardrails must be in place. You have been tasked with:

- Providing users with a Virtual Machine Contributor role to allow self-service provisioning of compute resources
- Enforcing mandatory `Environment`, `CostCenter`, and `Department` tags using Azure Policy
- Protecting shared resources against accidental deletion using resource locks
- Understanding Terraform's language constructs and how they map to Azure resources through the AzureRM provider
- Authoring a Terraform configuration from scratch that uses variables with validation, locals, data sources, resources, and outputs
- Deploying a baseline storage account using Terraform via Cloud Shell, with separate `.tfvars` files and workspaces for dev and prod
- Running a `terraform plan` to validate changes before applying them

## Architecture Overview

```
Tenant Root Group
└── Root Management Group
    └── [Your Subscription]
        ├── RG-Lab1 (Resource Group)
        │   ├── Azure Policy Assignments (3 tag policies)
        │   ├── Resource Lock (delete lock)
        │   └── Storage Account (deployed via Terraform)
```

## Job Skills

- Task 1: Configure RBAC with least-privilege role assignment
- Task 2: Enforce governance with Azure Policy and resource tags
- Task 3: Protect resources with a resource lock
- Task 4: Understand Terraform language constructs
- Task 5: Author a Terraform configuration from scratch
- Task 6: Deploy Terraform via Cloud Shell with environment variable files
- Task 7: Apply deployment best practices — plan, `.tfvars`, and state

---

## Task 1: Configure RBAC with Least-Privilege Role Assignment

Azure RBAC controls who can do what at which scope. In this task you assign a built-in role to an individual at the resource group scope.

1. Navigate to **RG-Lab1** and select **Access control (IAM)**.

2. Select the **Check access** tab and click **View my access** to see your current permissions on the resource group.

3. Select **Check access** button and in the **Check access** dialog search for any of your colleagues' user account. Notice that they have no access to the resource group.

4. Select the **Roles** tab and browse the list. Locate the **Virtual Machine Contributor** role and select **View under Details** to review the permissions it grants.

5. Visit Microsoft Learn: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles and find the **Virtual Machine Contributor** role. Review the list of permissions it grants.

6. Return to the portal and select **+ Add** → **Add role assignment**.

7. Search for and select **Virtual Machine Contributor**. Select **Next**.

8. On the **Members** tab, select **+ Select members**. Search for your colleague's user account. Select **Select**.

9. Select **Review + assign** twice.

10. Go to the **Role assignments** tab. Confirm the recent assignment appears on the **Role assignments** tab.

Best practices for RBAC:
- Assign roles to groups, not individuals — this simplifies management and ensures access is automatically updated as users join or leave groups.
- Use the narrowest scope that satisfies the requirement — assign at the resource group level instead of subscription if access is only needed for that resource group.
- Use built-in roles where possible, but create custom roles to remove permissions that built-in roles include but your scenario does not require (e.g. if Virtual Machine Contributor includes permissions you do not want to grant, create a custom role that excludes those permissions).

---

## Task 2: Enforce Governance with Azure Policy and Resource Tags

Azure Policy lets you enforce standards automatically. In this task you assign a built-in policy that requires a tag on all resources, then test that it blocks non-compliant deployments.

1. In the Azure portal, search for and select **Policy**.

2. In the **Authoring** blade, select **Definitions**.

3. Search for `Require a tag on resources`. Select the policy and review its definition — note the `deny` effect, which blocks non-compliant resources at deployment time.

4. Select **Assign policy**.

5. Set the **Scope** by selecting the ellipsis (…):

   | Setting | Value |
   | --- | --- |
   | Resource Group | **RG-Lab1** |

   Select **Select**.

6. Select **Next** and set **Parameters**:

   | Setting | Value |
   | --- | --- |
   | Tag Name | `Environment` |

7. On the **Non-compliance messages** tab, enter "Require Environment Tag on Resource".

8. Select **Review + Create**, then **Create**.

9. Repeat the **Assign policy** steps (Steps 4-8) two more times to enforce additional required tags:

   **Second assignment – Cost Center:**
   | Setting | Value |
   | --- | --- |
   | Tag Name | `CostCenter` |
   | Non-compliance message | `Require CostCenter Tag on Resource` |

   **Third assignment – Department:**
   | Setting | Value |
   | --- | --- |
   | Tag Name | `Department` |
   | Non-compliance message | `Require Department Tag on Resource` |

**Note:**
- The built-in policy `Require a tag on resources` enforces one tag per assignment. Assigning it once per required tag is the standard approach. Alternatively, use a Policy Initiative (policy set) to group all tag requirements into a single assignment.
- Policy enforcement typically takes 5–10 minutes to activate.

### Test the policy enforcement

1. Search for and select **Storage accounts**. Select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab1** |
   | Storage account name | `stlabyourname` |
   | Region | **Australia East** |
   | Primary Service | **Azure Blob Storage or Azure Data Lake Storage Gen 2** |
   | Redundancy | **LRS** |

3. **Do not add any tags.** Select **Review + Create**.

4. You should receive a **Validation failed** error. Select the Tags tab and confirm the error messages reference the missing tags.

**What just happened:** The ARM control plane evaluated the deployment request against all assigned policies before provisioning any resource. The `deny` effect caused the deployment to be rejected before anything was created.

5. In the **Tags** tab add all three required tags:

   | Name | Value |
   | --- | --- |
   | Environment | `dev` |
   | CostCenter | `CC1234` |
   | Department | `IT` |

6. Select **Review + Create**. This time validation passes.

7. Select **Create** and wait for the deployment to succeed.

8. Go to the Resource Group, expand the **Settings** blade and select **Policies**. You can see the compliance state of the Resource Group.

---

## Task 3: Protect Resources with a Resource Lock

Resource locks prevent accidental deletion or modification, independent of RBAC permissions. An Owner can be blocked from deleting a resource if a lock is present.

1. Navigate to **RG-Lab1**.

2. In the **Settings** blade, select **Locks**.

3. Select **+ Add** and configure:

   | Setting | Value |
   | --- | --- |
   | Lock name | `resourcegroup-nodelete` |
   | Lock type | **Delete** |
   | Notes | `Protects lab resource group from accidental deletion` |

4. Select **OK**.

5. Return to the **Overview** blade. Select **Delete resource group**.

6. Copy your resource group name and paste it in the confirmation field and select **Delete**.

7. You should receive a deletion failure notification. Confirm the error references the lock.

**Key point:** Resource locks operate at the Azure control plane layer and apply regardless of user permissions. A delete lock prevents removal; a read-only lock prevents both modifications and deletions. Locks must be explicitly removed before a resource can be deleted — this adds a deliberate friction step.

---

## Task 4: Understand Terraform Language Constructs

Terraform is a cloud-agnostic Infrastructure as Code tool from HashiCorp. You describe the desired state of your infrastructure in HashiCorp Configuration Language (HCL), and Terraform calls the Azure APIs through the **AzureRM provider** to make reality match your configuration. Terraform keeps a **state file** that records the resources it manages.

### Terraform concepts at a glance

| Concept | Syntax | Purpose |
| --- | --- | --- |
| Provider | `provider "azurerm" { features {} }` | Plugin that calls the Azure APIs |
| Input variable | `variable "name" { type = string }` | Value supplied at run time |
| Local value | `locals { sku = "LRS" }` | Value computed from variables |
| Data source | `data "azurerm_resource_group" "rg" { ... }` | Read an existing resource |
| Resource | `resource "azurerm_storage_account" "sa" { ... }` | Resource that Terraform creates and manages |
| String interpolation | `"prefix${var.name}"` | Combine strings and values |
| Conditional expression | `var.environment == "prod" ? "GRS" : "LRS"` | Choose a value based on a condition |
| Output | `output "id" { value = azurerm_storage_account.sa.id }` | Value returned after apply |
| Variable file | `terraform plan -var-file=dev.tfvars` | Environment-specific variable values passed with `-var-file` |
| Initialize | `terraform init` | Download the providers the configuration needs |
| Preview changes | `terraform plan` | Show what an apply would change |
| Apply changes | `terraform apply` | Create, update, or delete resources to match the configuration |
| State | `terraform.tfstate` | Record of the resources Terraform manages |

### Terraform language constructs

**Terraform and provider blocks** declare which Terraform version and providers the configuration needs. The `azurerm` provider requires an empty `features {}` block:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**Variables** accept values at run time. Arguments such as `description`, `type`, `default`, and `validation` add metadata and validation:

```hcl
variable "environment" {
  description = "Deployment environment."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be one of: dev, test, prod."
  }
}

variable "storage_account_name" {
  description = "Base name for the storage account."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{3,11}$", var.storage_account_name))
    error_message = "storage_account_name must be 3 to 11 lowercase letters or digits."
  }
}
```

**Locals** compute values from variables. Terraform supports ternary expressions and string interpolation:

```hcl
locals {
  # Ternary: choose replication type based on environment
  replication_type = var.environment == "prod" ? "GRS" : "LRS"

  # String interpolation: combine strings
  full_name = "${var.storage_account_name}${var.environment}"
}
```

**Data sources** read existing resources that Terraform does not manage. In this lab, the resource group (with its policies and lock) already exists:

```hcl
data "azurerm_resource_group" "lab" {
  name = var.resource_group_name
}
```

**Resources** use a resource type, a local name, and an arguments block. You reference other resources by `<type>.<name>.<attribute>`:

```hcl
resource "azurerm_storage_account" "storage" {
  name                     = local.full_name
  resource_group_name      = data.azurerm_resource_group.lab.name
  location                 = data.azurerm_resource_group.lab.location
  account_tier             = "Standard"
  account_replication_type = local.replication_type
  account_kind             = "StorageV2"

  # Block anonymous public access to blobs
  allow_nested_items_to_be_public = false
}
```

**Outputs** return values after `terraform apply` completes. You reference resource attributes directly using `<type>.<name>.<attribute>`:

```hcl
output "storage_account_name" {
  value = azurerm_storage_account.storage.name
}

output "storage_account_id" {
  value = azurerm_storage_account.storage.id
}

output "blob_endpoint" {
  value = azurerm_storage_account.storage.primary_blob_endpoint
}
```

### How the constructs connect

```
variable "storage_account_name" = "mystore"   // input
variable "environment"          = "dev"       // input
          ↓
local full_name        = "mystoredev"         // computed
local replication_type = "LRS"                // computed (env ≠ "prod")
          ↓
resource azurerm_storage_account.storage      // deployed and recorded in state
          ↓
output storage_account_id                     // returned
```

Terraform builds a dependency graph from these references, so it knows to read the resource group (data source) before creating the storage account.

---

## Task 5: Author a Terraform Configuration from Scratch

In this task you write a complete Terraform configuration that deploys a tagged storage account, using all the constructs from Task 4.

### Open Cloud Shell

1. Select the **Cloud Shell** icon in the top-right of the Azure portal.

2. When prompted, select **Bash**.

3. On the **Getting started** screen, select **No storage account required**, and select **Apply**. It may take a minute for the shell to provision.

4. Verify your Terraform version:

   ```bash
   terraform version
   ```
   Terraform is pre-installed in Cloud Shell and is updated to the latest release within a couple of weeks of its release. The output should show v1.5 or later. If a message says your version is **out of date**, it only means a newer release exists, and you can continue with the version shown. If the version is older than v1.5, see [Configure Terraform in Azure Cloud Shell with Bash](https://learn.microsoft.com/en-us/azure/developer/terraform/get-started-cloud-shell-bash).

5. Confirm you are using the correct subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

   If not, set it:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

6. Terraform in Cloud Shell authenticates automatically with your signed-in account. The AzureRM provider also needs to know which subscription to use, so export it as an environment variable:

   ```bash
   export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
   ```

### Create the Terraform files

1. Create a working folder:

   ```bash
   mkdir tf-lab1 && cd tf-lab1
   ```

2. Open the Cloud Shell editor:

   ```bash
   code main.tf
   ```

3. Paste in the following Terraform configuration:

```hcl
# ── Terraform and provider ────────────────────────────────────────────────────

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# ── Variables ─────────────────────────────────────────────────────────────────

variable "resource_group_name" {
  description = "Name of the existing resource group (for example RG-Lab1-yourname)."
  type        = string
}

variable "storage_account_name" {
  description = "Base name for the storage account (3–11 lowercase letters or digits)."
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9]{3,11}$", var.storage_account_name))
    error_message = "storage_account_name must be 3 to 11 lowercase letters or digits."
  }
}

variable "environment" {
  description = "Deployment environment."
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be one of: dev, test, prod."
  }
}

variable "cost_center" {
  description = "Cost center code for billing."
  type        = string
}

variable "department" {
  description = "Department that owns this resource."
  type        = string
}

variable "location" {
  description = "Azure region for all resources. Defaults to the resource group location."
  type        = string
  default     = null
}

# ── Data sources ──────────────────────────────────────────────────────────────

# The resource group (with its policies and lock) already exists, so read it instead of creating it
data "azurerm_resource_group" "lab" {
  name = var.resource_group_name
}

# ── Locals ────────────────────────────────────────────────────────────────────

locals {
  # Use GRS replication for production; LRS is sufficient for non-prod
  replication_type = var.environment == "prod" ? "GRS" : "LRS"

  # Append a deterministic hash so the name is globally unique, then cap at 24 characters
  # (storage_account_name ≤11 + environment ≤4 + hash 13 = up to 28; substr() enforces the limit)
  full_storage_name = substr(
    lower("${var.storage_account_name}${var.environment}${md5(data.azurerm_resource_group.lab.id)}"),
    0,
    24
  )

  location = coalesce(var.location, data.azurerm_resource_group.lab.location)

  # Centralise tags so every resource shares the same values
  common_tags = {
    Environment = var.environment
    CostCenter  = var.cost_center
    Department  = var.department
  }
}

# ── Resources ─────────────────────────────────────────────────────────────────

resource "azurerm_storage_account" "storage" {
  name                       = local.full_storage_name
  resource_group_name        = data.azurerm_resource_group.lab.name
  location                   = local.location
  account_tier               = "Standard"
  account_replication_type   = local.replication_type
  account_kind               = "StorageV2"
  access_tier                = "Hot"
  min_tls_version            = "TLS1_2"
  https_traffic_only_enabled = true
  tags                       = local.common_tags

  # Block anonymous public access to blobs
  allow_nested_items_to_be_public = false
}

# ── Outputs ───────────────────────────────────────────────────────────────────

output "storage_account_name" {
  value = azurerm_storage_account.storage.name
}

output "storage_account_id" {
  value = azurerm_storage_account.storage.id
}

output "blob_endpoint" {
  value = azurerm_storage_account.storage.primary_blob_endpoint
}
```

4. Use **Ctrl+S** to save, **Ctrl+Q** to close the editor.

### Understand what each section does

| Section | Purpose in this file |
| --- | --- |
| **Terraform and provider** | Pin the Terraform version and the `azurerm` provider version so every run behaves the same way. |
| **Variables** | Accept `resource_group_name`, `storage_account_name`, `environment`, `cost_center`, `department`, and `location` at run time. `validation` blocks add checks and documentation. |
| **Data sources** | Read the existing resource group so its name and location can be reused without hardcoding. |
| **Locals** | Compute the replication type (GRS for prod, LRS otherwise), build the globally-unique storage name, and assemble the shared tag map. |
| **Resources** | Deploy a `StorageV2` storage account using the computed values. Local name `storage` is used in outputs. |
| **Outputs** | Return the final storage account name, resource ID, and blob endpoint after apply. Terraform resolves these from the resource reference. |

**Key Terraform characteristics demonstrated:**
- `description`, `type`, and `validation` blocks make variables self-documenting and self-validating
- String interpolation `"${var.name}"` and the ternary expression `x ? a : b` keep logic readable
- Resource references such as `azurerm_storage_account.storage.id` create implicit dependencies automatically
- A data source reads an existing resource group without taking ownership of it
- The `tags` argument is sent when the storage account is created, so it satisfies the tag policies from Task 2

---

## Task 6: Deploy Terraform via Cloud Shell with Environment Variable Files

In this task you deploy the configuration authored in Task 5 using environment-specific variable files, then verify idempotency.

### Create environment variable files

Terraform supports `.tfvars` files that hold the values for your input variables. Use one file per environment so the same code is reused with different values.

1. Open the Cloud Shell editor to create the dev variable file:

   ```bash
   code dev.tfvars
   ```

2. Paste the following, replacing `RG-Lab1-yourname` with your resource group name and `abc` with 1–3 characters of your initials:

   > **Note:** `storage_account_name` must be **3 to 11 lowercase letters or digits** (enforced by the `validation` block). The prefix `stlabdev` is 8 characters, so you have room for up to 3 characters of your initials (e.g. `stlabdevjsm`).

   ```hcl
   resource_group_name  = "RG-Lab1-yourname"
   storage_account_name = "stlabdevabc"
   environment          = "dev"
   cost_center          = "CC1234"
   department           = "IT"
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

3. Create the prod variable file:

   ```bash
   code prod.tfvars
   ```

4. Paste the following, again replacing `RG-Lab1-yourname` with your resource group name and `abc` with the same 1–3 character initials you used above:

   > **Note:** `stlabprod` is 9 characters, so you have room for up to 2 characters of initials (e.g. `stlabprodjs`).

   ```hcl
   resource_group_name  = "RG-Lab1-yourname"
   storage_account_name = "stlabprodabc"
   environment          = "prod"
   cost_center          = "CC1234"
   department           = "IT"
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

**Why `.tfvars` files and workspaces?**
- Variable names and values are plain text, separate from the code, so the same `main.tf` serves every environment.
- Terraform tracks what it has deployed in a **state file**. If you applied `dev.tfvars` and then `prod.tfvars` against the same state, Terraform would see a different storage account name and try to *replace* the dev account. **Workspaces** give each environment its own state so dev and prod can coexist.
- Files named `terraform.tfvars` or `*.auto.tfvars` load automatically; other names, such as `dev.tfvars`, are passed explicitly with `-var-file`.

### Deploy using the dev variable file

1. Initialize the working directory. This downloads the `azurerm` provider:

   ```bash
   terraform init
   ```

   Confirm the output says `Terraform has been successfully initialized!`.

2. Create and switch to a `dev` workspace:

   ```bash
   terraform workspace new dev
   ```

3. Preview the changes:

   ```bash
   terraform plan -var-file=dev.tfvars
   ```

   Confirm the plan ends with `Plan: 1 to add, 0 to change, 0 to destroy.`

4. Apply the configuration:

   ```bash
   terraform apply -var-file=dev.tfvars
   ```

5. Type `yes` when prompted, then wait for the output to show `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`

6. Inspect the outputs returned by the deployment:

   ```bash
   terraform output
   ```

   Confirm `storage_account_name`, `storage_account_id`, and `blob_endpoint` are all present.

7. List storage accounts to confirm the new resource:

   ```bash
   az storage account list \
     --resource-group RG-Lab1-yourname \
     --query "[].{Name:name, SKU:sku.name, Location:location}" \
     -o table
   ```

### Verify idempotency

1. Run the exact same apply again:

   ```bash
   terraform apply -var-file=dev.tfvars
   ```

2. Confirm the output shows `No changes. Your infrastructure matches the configuration.` and no resource is created or modified.

**Idempotency** means running the same deployment twice produces the same end state without duplicating or erroring on existing resources. This is a core IaC principle — your Terraform runs should be safe to repeat at any time. Terraform achieves this by comparing your configuration, the state file, and the real Azure resources.

### Deploy using the prod variable file

1. Create and switch to a `prod` workspace:

   ```bash
   terraform workspace new prod
   ```

2. Apply the prod configuration, typing `yes` when prompted:

   ```bash
   terraform apply -var-file=prod.tfvars
   ```

3. Confirm success, then list storage accounts and verify the prod account has `Standard_GRS` SKU while the dev account has `Standard_LRS`.

   ```bash
   az storage account list \
     --resource-group RG-Lab1-yourname \
     --query "[].{Name:name, SKU:sku.name}" \
     -o table
   ```

   This confirms the ternary local `replication_type` correctly selected a different SKU for prod.

---

## Task 7: Apply Deployment Best Practices — Plan, Variables, State

### Plan: validate before you apply

`terraform plan` shows exactly what an apply *would* change without making any changes. Use this before every production deployment.

1. In Cloud Shell, switch back to the dev workspace and confirm it is selected:

   ```bash
   terraform workspace select dev
   terraform workspace show
   ```

2. Open your dev variable file to make a change:

   ```bash
   code dev.tfvars
   ```

3. Change the `cost_center` value to trigger an in-place update in the plan output:

   ```hcl
   resource_group_name  = "RG-Lab1-yourname"
   storage_account_name = "stlabdevabc"
   environment          = "dev"
   cost_center          = "CC9999"
   department           = "IT"
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

4. Run a plan:

   ```bash
   terraform plan -var-file=dev.tfvars
   ```

5. Review the output. Notice the change indicators:
   - **`+` create** — new resource will be added
   - **`~` update in-place** — existing resource will be changed
   - **`-` destroy** — resource will be removed
   - **`-/+` replace** — resource will be destroyed and recreated
   - Resources that are already in the desired state are not listed

   Here you should see the storage account marked `~` with `tags.CostCenter` changing from `"CC1234"` to `"CC9999"`, and `Plan: 0 to add, 1 to change, 0 to destroy.`

   > **Note:** Changing an argument that cannot be updated in place, such as the storage account `name`, produces a `-/+` replacement. In this lab the delete lock from Task 3 would block that destroy, which is a good reason to always read the plan before applying.

6. Only proceed with `terraform apply` once you are satisfied with the plan output:

   ```bash
   terraform apply -var-file=dev.tfvars
   ```

7. In the portal, open the dev storage account and confirm the `CostCenter` tag now shows `CC9999`.

### Formatting and validation with the Terraform CLI

Terraform includes built-in commands that catch common authoring mistakes before deployment. `terraform fmt` checks formatting and `terraform validate` checks that the configuration is syntactically valid and internally consistent. In this section you first run them against a file with intentional issues to see what errors look like, then run them against `main.tf` to confirm a clean result.

#### Create a file with intentional issues

1. Create a separate folder so it does not affect your lab configuration:

   ```bash
   cd ~ && mkdir witherrors && cd witherrors
   ```

2. Open the Cloud Shell editor and create `main.tf`:

   ```bash
   code main.tf
   ```

3. Paste in the following file, which contains deliberate problems:

   ```hcl
   # This file contains intentional issues for demonstration purposes.

   variable "storage_account_name" {
   type = string
   description = "Base name for the storage account."
   }

   locals {
   # Issue 1: Inconsistent indentation and alignment (terraform fmt)
   suffix = "dev"

   # Issue 2: Reference to an undeclared variable (terraform validate)
   storage_name = "${var.storage_name}${local.suffix}"
   }

   # Issue 3: Reference to an undeclared local value (terraform validate)
   output "storage_name" {
     value = local.full_name
   }
   ```

   Use **Ctrl+S** to save, **Ctrl+Q** to close.

#### Run the checks against the file with errors

1. Check formatting:

   ```bash
   terraform fmt -check -diff
   ```

   The command lists `main.tf` as needing changes and shows a diff of the indentation and alignment fixes it would make.

2. Initialize and validate the configuration:

   ```bash
   terraform init -backend=false
   terraform validate
   ```

3. You should see output similar to:

   ```
   ╷
   │ Error: Reference to undeclared input variable
   │
   │   on main.tf line 12, in locals:
   │   12:   storage_name = "${var.storage_name}${local.suffix}"
   │
   │ An input variable with the name "storage_name" has not been declared. Did you mean "storage_account_name"?
   ╵
   ╷
   │ Error: Reference to undeclared local value
   │
   │   on main.tf line 17, in output "storage_name":
   │   17:   value = local.full_name
   │
   │ A local value with the name "full_name" has not been declared.
   ╵
   ```

4. Each error shows the **file and line**, the offending code, and a message that often includes a suggested fix. Note:
   - `terraform fmt` is about style; it exits with a non-zero code when files need changes, so it can be used in a pipeline.
   - `terraform validate` is about correctness; it does not contact Azure and does not need credentials.
   - Run `terraform fmt` (without `-check`) to fix formatting automatically.

#### Run the checks against your lab configuration

1. Return to the lab folder:

   ```bash
   cd ~/tf-lab1
   ```

2. Run:

   ```bash
   terraform fmt -check
   terraform validate
   ```

3. `terraform fmt -check` prints nothing, and `terraform validate` reports `Success! The configuration is valid.` This means the files passed both checks.

### Review state and deployment history

1. List the resources Terraform manages in the dev workspace:

   ```bash
   terraform state list
   ```

2. Inspect the recorded attributes of the storage account:

   ```bash
   terraform state show azurerm_storage_account.storage
   ```

   Confirm the tags, replication type, and resource ID are all recorded.

3. List the workspaces and confirm that dev and prod each have their own state:

   ```bash
   terraform workspace list
   ```

4. In the Azure portal, navigate to **RG-Lab1** and select **Activity log**. Review the operations Terraform performed — each entry records who made the change, when, and the outcome.

**Common pitfalls to avoid:**
- Hardcoded resource names in Terraform files — use `variable` instead
- Missing `validation` blocks on environment variables — allows invalid values to slip through
- Skipping `terraform plan` before production applies
- Not committing `.tf` and `.tfvars` files to source control — they are part of your IaC and should be versioned
- Committing the state file or secrets to source control — state can contain sensitive values; use a remote backend such as an Azure Storage account for team work
- Editing Terraform-managed resources in the Azure portal — creates drift that the next `terraform plan` will try to undo
- Running Terraform in Cloud Shell with **No storage account required** — local state is lost when the session ends, so use a remote backend for anything beyond this lab

---

## Cleanup

**Note:** Remove the resource lock before deleting the resource group, otherwise the deletion will be blocked.

1. Go to your Resource Group.

2. In the **Settings** → **Locks**, select the lock and click **Delete**.

3. In the **Overview**, select **Delete resource group**.

4. Copy and paste the name to confirm deletion.

5. Select **Delete**.

---

## Key Takeaways

- **RBAC least privilege**: assign roles to groups, not individuals; use the narrowest scope that satisfies the requirement; use built-in roles as much as possible.

- **Azure Policy** enforces governance standards at the control plane before resources are created. The `deny` effect prevents non-compliant deployments; the `modify` effect with a remediation task corrects existing resources.

- **Resource locks** protect critical resources from accidental deletion or modification regardless of RBAC permissions — they add deliberate friction.

- **Terraform** is a multi-cloud Infrastructure as Code tool. It uses providers such as `azurerm` to call Azure APIs and tracks the resources it manages in a state file. It offers a consistent workflow (`init`, `plan`, `apply`) across clouds.

- **Variables with validation** (`description`, `type`, `validation`) make Terraform configurations self-documenting and self-validating without extra tooling.

- **Ternary expressions and string interpolation** keep logic in locals readable and replace repeated hardcoded values.

- **Resource references** such as `azurerm_storage_account.storage.id` let Terraform resolve attributes directly, and enable it to build the dependency graph and infer ordering automatically.

- **Data sources** read existing resources, such as the lab resource group, without managing them.

- **`.tfvars` files and workspaces** let one configuration serve several environments, each with its own variable values and its own state.

- **Idempotency** means the same configuration can be safely re-applied at any time and will converge to the same result without duplicating resources.

- **`terraform plan`** is a mandatory step before production applies — it shows exactly what will change before anything is applied. `terraform fmt` and `terraform validate` catch mistakes earlier still.

---

## Resources

- [Terraform on Azure documentation](https://learn.microsoft.com/en-us/azure/developer/terraform/)
- [Configure Terraform in Azure Cloud Shell with Bash](https://learn.microsoft.com/en-us/azure/developer/terraform/get-started-cloud-shell-bash)
- [Authenticate Terraform to Azure](https://learn.microsoft.com/en-us/azure/developer/terraform/authenticate-to-azure)
- [Store Terraform state in Azure Storage](https://learn.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage)
- [Terraform AzureRM provider documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform language documentation](https://developer.hashicorp.com/terraform/language)
- [Azure Governance documentation](https://learn.microsoft.com/en-us/azure/governance/)
- [Azure RBAC built-in roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles)
- [Azure Policy built-in definitions](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies)
