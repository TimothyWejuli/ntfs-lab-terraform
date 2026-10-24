# NTFS File Server Lab

Azure lab for Windows Server administration: Active Directory, NTFS access control, SMB file shares, and Group Policy. Deployed with Terraform and configured with PowerShell through Azure Run Command.

## What You'll Learn

- Deploying Windows Server infrastructure on Azure with Terraform
- Keeping the VM admin password in Azure Key Vault (RBAC authorization)
- Storing Terraform state remotely in Azure Blob Storage
- Promoting a server to an Active Directory domain controller
- Configuring VMs without RDP by using `az vm run-command`
- Creating SMB shares and applying NTFS permissions
- Controlling access with security groups instead of individual users
- Creating a Group Policy Object for RDP access
- Joining Windows servers and a Windows 11 client to a domain

## Architecture

| VM | Role | OS |
|----|------|----|
| DC01 | Domain controller (static IP 10.0.1.4) | Windows Server 2022 |
| FS01 | File server | Windows Server 2022 |
| CLIENT01 | Client workstation | Windows 11 Pro |

- **Domain:** `lab.local`
- **Region:** Central US
- **VM size:** Standard_B2as_v2
- **Network:** VNet 10.0.0.0/16 with one subnet 10.0.1.0/24; an NSG allows RDP (3389) only from your public IP
- **Secrets:** Azure Key Vault holds the VM admin password
- **State:** Terraform state is stored in Azure Blob Storage

> **Cost estimate:** Running all three VMs costs roughly $0.15 to $0.25 per hour. Delete the resource group when you finish to avoid charges.

## Prerequisites

| Tool | Version |
|------|---------|
| Terraform | 1.5.0 or later |
| Azure CLI | Recent version |
| Azure subscription | Active |
| Windows PowerShell | 5.1 or later |

## Step 1: Configure Variables

```powershell
Copy-Item terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars` and set `rdp_source` to your public IP in CIDR form (for example `1.2.3.4/32`). Set the admin password as an environment variable so it never goes in a file:

```powershell
$env:TF_VAR_admin_password = "<strong password>"
```

## Step 2: Deploy Infrastructure

```powershell
az login
terraform init
terraform plan
terraform apply      # about 10-15 minutes
terraform output key_vault_name
```

## Step 3: Configure the Lab (Automated)

```powershell
.\configure-lab.ps1 -KeyVaultName "<key_vault_name>"
```

Total time is about 15 to 20 minutes, unattended. A `[PASS]` / `[FAIL]` report prints at the end.

## Step 4: Verify the Lab

RDP to **CLIENT01**, sign in as a domain user such as `LAB\sarah.jones`, then check identity and name resolution:

```powershell
whoami
nslookup FS01.lab.local
```

Test access with `Get-ChildItem` (read) and `New-Item` (write). `Test-Path` alone does not reliably show permissions.

As sarah.jones (GRP_Finance), the Finance share accepts a new file, while reading the HR share is denied:

<img width="1003" height="684" alt="Screenshot-2026-10-22-165126" src="https://github.com/user-attachments/assets/79e627d2-4f07-475a-9659-1568d7fbdced" />

| User | Share | Expected result |
|------|-------|-----------------|
| sarah.jones (GRP_Finance) | `\\FS01\Finance` | Read and write |
| sarah.jones (GRP_Finance) | `\\FS01\HR` | Access denied |
| lisa.white (GRP_HR) | `\\FS01\Finance` | Read only |
| lisa.white (GRP_HR) | `\\FS01\HR` | Read and write |
| john.smith (GRP_IT) | `\\FS01\IT` | Full control |
| tom.davis (GRP_Sales) | `\\FS01\Finance` | Access denied |

Share-level permission is Everyone: Full Control by design. NTFS permissions do the enforcing.

Running `icacls` against the HR share as a Finance user is also refused, which confirms the NTFS permissions are doing the blocking:

<img width="523" height="84" alt="Icacls" src="https://github.com/user-attachments/assets/773fc1c6-60f7-4701-be3c-243cc0df9eed" />

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `configure-lab.ps1` fails at Key Vault | Run `az login` again and confirm your account has a Key Vault secrets role on the vault |
| Domain join fails | DC01 is likely still finishing promotion; wait 2 minutes and rerun |
| Cannot RDP to a VM | Check that `rdp_source` matches your current public IP, then run `terraform apply` |
| Access denied unexpectedly | Run `whoami /groups` to confirm group membership, then sign out and back in |
| `terraform apply` prompts for variables | `terraform.tfvars` is missing or misnamed; copy it from the example file |
| `terraform init` fails | Confirm the storage account name in `backend.tf` and that the `tfstate` container exists |

## Step 5: Stop the VMs

If you continue to Part 2, do not destroy the resource group. Deallocate the VMs instead:

```powershell
az vm deallocate --ids $(az vm list -g RG-FileServerLab --query "[].id" -o tsv) --no-wait
```

## Part 2: Azure RBAC Access Control

Part 2 builds directly on this infrastructure. It adds Azure RBAC role assignments (Owner, Virtual Machine Contributor, and Reader) scoped to the FS01 virtual machine, so you control who can manage the server from Azure in addition to who can reach its files.

- **Repository:** [azure-rbac-lab-terraform](https://github.com/TimothyWejuli/azure-rbac-lab-terraform)
- **Requires from this lab:** the `RG-FileServerLab` resource group, the FS01 VM, and the `RG-TerraformState` storage account
- **Creates:** three role assignments only, with no new VMs or networking

> **Before you start Part 2:** keep this lab's resource group and VMs in place. Deallocate them (Step 5) instead of deleting them, then start the VMs again and confirm all three show `VM running`.

## Teardown

When both parts are finished, run Part 2's `terraform destroy` first, then delete this lab's resource group:

```powershell
az group delete -n RG-FileServerLab --yes --no-wait
```
