# Azure DevOps Pipeline + Terraform Deployment Tutorial

![None](https://miro.medium.com/v2/resize\:fit:700/0*5iGW5XrxHIeZb-eX.png)

## Use Azure DevOps to Automate Terraform Deployments in Azure

In this walkthrough, we’ll build a robust multi-stage Azure DevOps pipeline that deploys Terraform-managed infrastructure to Azure. Along the way, we’ll:

* Store Terraform backend state in Azure Blob Storage
* Configure a Service Principal and grant it scoped RBAC access
* Build a parameterized multi-stage pipeline with manual approvals
* Show how this setup can scale for multiple environments

---

### Topology Overview

The diagram below outlines the high-level components:

![None](https://miro.medium.com/v2/resize\:fit:700/0*5iGW5XrxHIeZb-eX.png)

Key parts to highlight:

* **ADO Build Agent**: We’re using a hosted Ubuntu agent with Terraform preinstalled.
* **ADO Git Repo**: Stores all pipeline YAML and Terraform code.
* **Terraform State**: Stored in Azure Blob Storage within a dedicated storage account.
* **Resource Group**: Terraform creates *env01-tfdemo-rg*, our deployment target.

File structure:

![None](https://miro.medium.com/v2/resize\:fit:700/0*HD9rHU2N71ryGAuW.png)

> Tip: Pipeline YAMLs can live anywhere, but we’ll keep ours in `/deploy`.

---

### Create the Service Principal (SPN)

To authenticate Terraform from ADO, we need a Service Principal:

1. Go to **Microsoft Entra ID** in the Azure Portal.
2. Create a new **App registration**: `tfdemo-spn`
3. Note the **Application (client) ID** and **Directory (tenant) ID**
4. Under **Certificates & Secrets**, generate a new secret named `ADO` and save its value.

---

### Set Up Terraform Backend in Azure Blob

Terraform stores state data centrally. We'll host that in Azure Blob Storage:

1. Create Resource Group: `tfstate-tfdemo-rg`
2. Create Storage Account: `tfstatedemostg`
3. Create Blob Container: `tfstate`

Grant the SPN **Storage Blob Data Contributor** on the storage account:

![None](https://miro.medium.com/v2/resize\:fit:700/0*M2hk7fQjuvsZFPMV.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*aNWzEt0OhbJ2I03U.png)

> No need to create the tfstate file manually—Terraform handles that on first run.

---

### Assign Azure RBAC Permissions

Terraform needs permission to create Azure resources:

1. Go to the Azure subscription → **Access Control (IAM)**
2. Assign the SPN the **Contributor** role on the subscription

![None](https://miro.medium.com/v2/resize\:fit:700/0*A4td9JJmFUWkEgdj.png)

> Best practice: limit SPN scope to a resource group instead of the entire subscription if possible.

---

### Terraform Code

Basic Service Bus deployment:

* One namespace
* Two queues

`servicebus.tf`:

```hcl
<same as original>
```

And in `providers.tf`, configure the remote backend:

```hcl
<same as original>
```

Full code: [GitHub Repo](https://github.com/andrew-kelleher/terraformadopipeline)

---

### Azure DevOps Configuration

#### Variable Group

Under **Pipelines → Library**, create a variable group: `Terraform_SPN` with these values:

* `ARM_CLIENT_ID`
* `ARM_CLIENT_SECRET`
* `ARM_SUBSCRIPTION_ID`
* `ARM_TENANT_ID`

Terraform will use these environment variables for authentication.

![None](https://miro.medium.com/v2/resize\:fit:700/0*m3N_RVVxm9WznDRg.png)

#### Environment & Approval

Add an ADO [Environment](https://techcommunity.microsoft.com/t5/healthcare-and-life-sciences/azure-devops-pipelines-environments-and-variables/ba-p/3707414) named `env01`, then configure **Approvals and Checks** with your user or team.

![None](https://miro.medium.com/v2/resize\:fit:700/0*lwx6MDC8kbvKvkoY.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*aVMcQ_NuQ6cXCWC3.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*Z1LsCoH66MgO41gZ.png)

#### 🛠️ Build the Pipeline

We’ll use a reusable YAML template.

`terraform-template.yml`:

```yaml
<same as original>
```

Then reference it in a separate pipeline YAML for `env01`:

`tfdemo-env01-terraform.yml`:

```yaml
<same as original>
```

Create pipeline in ADO:

1. New Pipeline → Azure Repos Git → select your repo
2. Point to `tfdemo-env01-terraform.yml`
3. Click Save

![None](https://miro.medium.com/v2/resize\:fit:700/0*yY-R-oUlT_HlVX7J.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*9o7gmDvrioFMgTh7.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*T5kb7_y4Q6Noot5l.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*y_2ke5BSfJ4SWxe4.png)

---

### Run the Pipeline

Trigger the pipeline manually.

After the **Terraform Plan** stage runs, review and approve the **Terraform Apply** step:

![None](https://miro.medium.com/v2/resize\:fit:700/0*PwxzGCYXsD6Gx4mt.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*1dUp-tbM2maDzGay.png)
![None](https://miro.medium.com/v2/resize\:fit:700/0*7LhlW8AthtNelYR8.png)

> On first run, permit access to the variable group and environment.

---

### Scale to Multi-Environment

You can reuse this template to deploy to DEV, UAT, and PROD. Simply do the following:

* Copying the pipeline YAML
* Creating a new tfvars file
* Updating the environment name

This allows you to replicate complete application environments across stages.

![None](https://miro.medium.com/v2/resize\:fit:700/0*BuyRRwrLmfHWVqnO.png)

---

### ✅ Wrapping Up

We’ve built a modular, secure CI/CD pipeline using Azure DevOps and Terraform. While this example used a simple Service Bus setup, it’s easy to extend the pipeline and Terraform code to deploy any Azure infrastructure component you need.
