# Azure RBAC Access Control Lab (Terraform + PowerShell)

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-7B42BC?logo=terraform&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-RBAC-0078D4?logo=microsoftazure&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-Validation-5391FE?logo=powershell&logoColor=white)
![Cost](https://img.shields.io/badge/Role%20assignments-Free-2ea44f)

**Least-privilege access control for an Azure file server, deployed as code and verified against the live control plane.**

Three personas, three roles, one VM. Every assignment is scoped to **FS01's resource ID only**. DC01 and CLIENT01 sit in the same resource group and get no access from this lab.

## 🎬 Watch Me Build This Lab!
https://www.loom.com/share/1d818897b1894f61a85ac8ffbac52a6b

| | |
|---|---|
| **Depends on** | [Lab 1: NTFS File Server](#how-this-lab-fits-into-the-series). `RG-FileServerLab` and `FS01` must exist |
| **What gets created** | 3 role assignments on FS01. No VMs, no networking, no resource groups |
| **Deploy time** | Under 1 minute (plus 2–3 min RBAC propagation) |
| **Cost** | Role assignments are free. Only the Lab 1 VMs incur compute charges |

---

## The Business Problem

Lab 1 controlled **who can access files inside the VM** (NTFS). This lab controls **who can manage the VM itself from Azure** (RBAC). Those are two separate layers. A file server can have perfect NTFS permissions and still be at risk if anyone with a portal login can stop or delete it.

> **Scenario:** A ticket comes in saying the Finance file server is unresponsive. The on-call **SupportTech** restarts it from Azure. They cannot delete it, resize it, or see who else has access. At quarter end, the **Auditor** exports a report confirming that only authorized people have access. After a staff change, the **SysAdmin** uses Owner rights to reassign roles. Each person has exactly the access their job requires and no more.

## Architecture

![Lab 2 Azure RBAC architecture](diagrams/lab2_rbac_architecture.png)

Azure RBAC operates at the **Resource Manager control plane**, which is completely separate from the NTFS permissions inside the VM's OS.

## Role Design

| Persona | Role | Scope | Can | Cannot |
|---|---|---|---|---|
| **SysAdmin** | Owner | FS01 only | Full control, including managing RBAC | n/a |
| **SupportTech** | Virtual Machine Contributor | FS01 only | Start, stop, restart, connect | Manage RBAC (can still *view* assignments) |
| **Auditor** | Reader | FS01 only | View configuration and status | Take any action |

### Permission Matrix (FS01 scope)

| Action | Owner | VM Contributor | Reader |
|---|:---:|:---:|:---:|
| View VM details | ✅ | ✅ | ✅ |
| Start / Stop VM | ✅ | ✅ | ❌ |
| Connect via RDP | ✅ | ✅ | ❌ |
| Delete VM | ✅ | ⚠️ *see note* | ❌ |
| Manage RBAC roles | ✅ | ❌ | ❌ |

> **Note:** It's easy to assume VM Contributor can't delete, and the original SOP did. The built-in *Virtual Machine Contributor* role actually includes `Microsoft.Compute/virtualMachines/*`, which covers delete. To truly block deletion for the help desk persona, use a custom role that excludes `virtualMachines/delete` or put a `CanNotDelete` resource lock on FS01. See [Lessons Learned](#lessons-learned).

## Repository Structure

```
lab-2-azure-rbac/
├── backend.tf                 # Reuses Lab 1 storage account, separate state key
├── versions.tf                # Terraform >= 1.5, azurerm ~> 3.0
├── variables.tf               # Three Object ID inputs with NO defaults (fail loud)
├── data.tf                    # Reads Lab 1 RG + FS01 without modifying them
├── rbac.tf                    # Three role assignments scoped to FS01's resource ID
├── outputs.tf                 # VM ID, RG name, role assignment IDs (sensitive)
├── terraform.tfvars.example   # Safe template, commit this
├── .gitignore                 # Keeps real tfvars, state, and reports out of git
├── validate-lab.ps1           # Queries live RBAC, prints PASS/FAIL + matrix, exports report
├── scripts/
│   └── 01-get-object-ids.ps1  # UPN → Entra ID Object ID lookup
├── diagrams/
│   └── lab2_rbac_architecture.png
└── docs/
    ├── SOP.md                 # Full step-by-step runbook
    └── Lab2_Azure_RBAC_SOP.docx
```

## Why Each Design Decision Matters

- **Scope = the VM's resource ID, not the resource group.** This is the most important line in each assignment. If it pointed at the RG, the role would silently extend to DC01, CLIENT01, and every other resource in `RG-FileServerLab`. A narrow scope keeps the blast radius small if an account is compromised.
- **Data sources, not resources, for Lab 1 infrastructure.** Terraform *reads* FS01 and never modifies it. This is the same pattern one team uses to reference another team's infrastructure.
- **No defaults on Object ID variables.** A careless `apply` can't assign roles to placeholder identities because Terraform fails loudly instead.
- **Object IDs, not email addresses.** UPNs can change but Object IDs never do. `01-get-object-ids.ps1` does the lookup.
- **Separate state key (`rbac-lab.terraform.tfstate`).** It shares Lab 1's storage account but not its state. If both labs used the same key, `terraform destroy` here would tear down Lab 1.
- **Validation runs separately from `terraform apply`.** Apply only confirms that Terraform created the resource. `validate-lab.ps1` confirms the permission actually propagated in Azure RBAC. Those are two different systems.

## Quick Start

### Prerequisites
```powershell
terraform -version   # >= 1.5.0
az version           # any recent Azure CLI
az account show      # confirm correct subscription

# Lab 1 VMs must be running
az vm list -g RG-FileServerLab --query "[].{name:name,status:powerState}" -o table
```

You also need **three test accounts in Entra ID** (not your own). If your tenant enforces MFA, use a Conditional Access exclusion or a test tenant so you can `az login` as each persona.

### Deploy
```powershell
# 1. Look up Object IDs for the three test users
.\scripts\01-get-object-ids.ps1

# 2. Configure variables
Copy-Item terraform.tfvars.example terraform.tfvars   # paste Object IDs in
# Edit backend.tf → replace REPLACE_WITH_YOUR_LAB1_STORAGE_ACCOUNT_NAME

# 3. Deploy
az login
terraform init
terraform plan    # Must show exactly 3 to add. Anything more = stop and check tfvars
terraform apply

# 4. Wait 2–3 minutes for propagation, then validate
.\validate-lab.ps1
```

### Test As Each Persona
Open a new terminal for each persona. The **expected failures show that least privilege is working**. They are not errors.

| Persona | Command | Expected |
|---|---|---|
| Auditor | `az vm show -g RG-FileServerLab -n FS01` | ✅ Succeeds |
| Auditor | `az vm stop -g RG-FileServerLab -n FS01` | ❌ `AuthorizationFailed` |
| SupportTech | `az vm start -g RG-FileServerLab -n FS01` | ✅ Succeeds |
| SupportTech | `az role assignment create ...` on FS01 | ❌ `AuthorizationFailed` |
| SysAdmin | `az role assignment list --scope <FS01 id>` | ✅ Succeeds |

Full commands are in [`docs/SOP.md`](docs/SOP.md#step-7--test-as-each-persona).

### Teardown
```powershell
terraform destroy                                   # Remove RBAC only; Lab 1 keeps running

# Done with both labs:
az group delete -n RG-FileServerLab --yes --no-wait
```

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `principal not found` on apply | Wrong or deleted Object ID | Re-run `01-get-object-ids.ps1`, update tfvars |
| `resource group not found` | Lab 1 not deployed | Deploy Lab 1 first |
| Plan shows more than 3 resources | RG or VM name mismatch | Names are case-sensitive: `FS01` ≠ `fs01` |
| Validate shows FAIL after 10+ min | Assignment missing from state | `terraform state list`, then re-apply |
| Auditor can still stop the VM | RBAC still propagating | Wait 5 min, retest |
| MFA prompt as test user | Tenant enforces MFA | Conditional Access exclusion or test tenant |

## How This Lab Fits Into the Series

| Lab | Deploys | Relationship |
|---|---|---|
| **Lab 1: NTFS File Server** | DC01, FS01, CLIENT01, VNet, NSG, Key Vault in `RG-FileServerLab` | Standalone |
| **Lab 2: Azure RBAC** *(this repo)* | 3 role assignments on FS01 | Reads Lab 1 via data sources; reuses Lab 1's state storage account |
| **AUM Lab: Azure Update Manager** | DC01, WS01, WS02, VNet, Key Vault in `rg-aumlab` | Fully independent |

## Lessons Learned

- **`apply` succeeding doesn't prove the permission is live.** RBAC propagation lags, so check against the control plane before you trust it.
- **Built-in roles are broader than their names suggest.** "VM Contributor" sounds like help desk access, but it can still delete the VM. In production, confirm the role's actions (`az role definition list --name "Virtual Machine Contributor"`) before assigning it.
- **Owner is more than most people need.** In production, Contributor plus User Access Administrator assigned separately is the more precise least-privilege split.
- **Next iteration:** a custom help desk role, a `CanNotDelete` lock on FS01, PIM for just-in-time Owner elevation, and assignments to Entra ID **groups** instead of individual users.

---

📄 Full runbook: [`docs/SOP.md`](docs/SOP.md)
