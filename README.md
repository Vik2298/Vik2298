# Azure Infrastructure with Terraform
 
This project provisions a complete Azure infrastructure using Terraform, including networking, security, and a Linux virtual machine with SSH access.
 
## Architecture
 
```
                        Internet
                           |
                    [ Public IP ]
                           |
                    [ Network Interface ] --- [ NSG (SSH + HTTP) ]
                           |
                    [ Subnet 1: 10.0.1.0/24 ]
                    [ Subnet 2: 10.0.2.0/24 ]
                           |
                    [ VNet: 10.0.0.0/16 ]
                           |
                    [ Resource Group ]
```
 
## Resources Created
 
| # | Resource | Name | Description |
|---|----------|------|-------------|
| 1 | Resource Group | `sxalable-rg-dev` | Container for all resources |
| 2 | Virtual Network | `sxalable-vnet-dev` | VNet with address space `10.0.0.0/16` |
| 3 | Subnet 1 | `sxalable-s1-dev` | Subnet with CIDR `10.0.1.0/24` |
| 4 | Subnet 2 | `sxalable-s2-dev` | Subnet with CIDR `10.0.2.0/24` |
| 5 | Public IP | `sxalable-pip-dev` | Static Standard SKU public IP with availability zone |
| 6 | Network Security Group | `sxalable-sg-dev` | NSG with SSH (22) and HTTP (80) inbound rules |
| 7 | Network Interface | `sxalable-ni1-dev` | NIC attached to Subnet 1 with Public IP |
| 8 | NIC-NSG Association | - | Links NIC to the NSG |
| 9 | Linux Virtual Machine | `sxalable-vm-dev` | Ubuntu 22.04 LTS VM with SSH key authentication |
 
## Prerequisites
 
- [Terraform](https://developer.hashicorp.com/terraform/downloads) >= 1.3.0
- An Azure account with an active subscription
- An Azure Service Principal with **Contributor** role
- An SSH key pair (`~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`)
 
## File Structure
 
```
Azure-TF/
  ├── providers.tf       # Terraform and AzureRM provider configuration
  ├── variables.tf       # Input variable declarations
  ├── main.tf            # All Azure resource definitions
  ├── outputs.tf         # Output values (public IP, SSH command, etc.)
  ├── terraform.tfvars   # Variable values (your credentials and settings)
  ├── .gitignore         # Ignores state files, .terraform/, tfvars
  └── README.md          # This file
```
 
## Setup
 
### 1. Create a Service Principal
 
```bash
az login
az ad sp create-for-rbac \
  --name "terraform-sp" \
  --role Contributor \
  --scopes /subscriptions/<YOUR_SUBSCRIPTION_ID>
```
 
This outputs:
- `appId` --> use as `client_id`
- `password` --> use as `client_secret`
- `tenant` --> use as `tenant_id`
 
### 2. Generate SSH Keys (if not already present)
 
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```
 
### 3. Configure Variables
 
Edit `terraform.tfvars` with your values:
 
```hcl
subscription_id  = "your-subscription-id"
client_id        = "your-client-id"
client_secret    = "your-client-secret"
tenant_id        = "your-tenant-id"
azure_region     = "Central US"
vm_size          = "Standard_D2s_v3"
zone             = "1"
public_key_path  = "~/.ssh/id_rsa.pub"
private_key_path = "~/.ssh/id_rsa"
```
 
### 4. Deploy
 
```bash
# Initialize Terraform (downloads the AzureRM provider)
terraform init
 
# Preview what will be created
terraform plan
 
# Create all resources
terraform apply -auto-approve
```
 
### 5. Connect to the VM
 
After deployment, Terraform outputs the SSH command:
 
```bash
ssh ubuntu@<PUBLIC_IP_ADDRESS>
```
 
### 6. Destroy Resources
 
```bash
terraform destroy -auto-approve
```
 
## Variables Reference
 
| Variable | Description | Default |
|----------|-------------|---------|
| `subscription_id` | Azure subscription ID | *(required)* |
| `client_id` | Service Principal client ID | *(required)* |
| `client_secret` | Service Principal client secret (sensitive) | *(required)* |
| `tenant_id` | Azure tenant ID | *(required)* |
| `azure_region` | Azure region for all resources | `East US` |
| `vm_size` | VM size / instance type | `Standard_B1s` |
| `zone` | Availability zone (1, 2, or 3) | `1` |
| `public_key_path` | Path to SSH public key file | `~/.ssh/id_rsa.pub` |
| `private_key_path` | Path to SSH private key file | `~/.ssh/id_rsa` |
 
## Outputs
 
| Output | Description |
|--------|-------------|
| `resource_group_name` | Name of the created resource group |
| `virtual_network_name` | Name of the virtual network |
| `subnet1_id` | Resource ID of Subnet 1 |
| `subnet2_id` | Resource ID of Subnet 2 |
| `public_ip_address` | Public IP address assigned to the VM |
| `vm_name` | Name of the virtual machine |
| `ssh_connection` | Ready-to-use SSH command |
 
## Troubleshooting
 
### SkuNotAvailable Error
 
If you get `SkuNotAvailable` when creating the VM, it means the selected VM size is not available in that region/zone. Try:
 
1. **Change the zone** in `terraform.tfvars`:
   ```hcl
   zone = "2"   # Try zones 1, 2, or 3
   ```
 
2. **Change the region**:
   ```hcl
   azure_region = "West US 2"
   ```
 
3. **Change the VM size**:
   ```hcl
   vm_size = "Standard_D2s_v3"   # D-series has better availability
   ```
 
### AuthorizationFailed Error
 
Your Service Principal needs **Contributor** role on the subscription:
 
```bash
az role assignment create \
  --assignee "<CLIENT_ID>" \
  --role "Contributor" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>"
```
 
## Azure VM Sizes - Max Network Interfaces (NICs)
 
Each Azure VM size supports a limited number of NICs. You must choose the right VM size based on how many NICs your workload needs.
 
### D-series v3 (General Purpose)
 
```
Standard_D2s_v3   ->  2 network interfaces
Standard_D4s_v3   ->  2 network interfaces
Standard_D8s_v3   ->  4 network interfaces
Standard_D16s_v3  ->  8 network interfaces
Standard_D32s_v3  ->  8 network interfaces
Standard_D48s_v3  ->  8 network interfaces
Standard_D64s_v3  ->  8 network interfaces
```
 
### D-series v5 (General Purpose - Latest Generation)
 
```
Standard_D2s_v5   ->  2 network interfaces
Standard_D4s_v5   ->  2 network interfaces
Standard_D8s_v5   ->  4 network interfaces
Standard_D16s_v5  ->  8 network interfaces
Standard_D32s_v5  ->  8 network interfaces
Standard_D48s_v5  ->  8 network interfaces
Standard_D64s_v5  ->  8 network interfaces
Standard_D96s_v5  ->  8 network interfaces
```
 
### E-series v5 (Memory Optimized)
 
```
Standard_E2s_v5   ->  2 network interfaces
Standard_E4s_v5   ->  2 network interfaces
Standard_E8s_v5   ->  4 network interfaces
Standard_E16s_v5  ->  8 network interfaces
Standard_E20s_v5  ->  8 network interfaces
Standard_E32s_v5  ->  8 network interfaces
Standard_E48s_v5  ->  8 network interfaces
Standard_E64s_v5  ->  8 network interfaces
Standard_E96s_v5  ->  8 network interfaces
```
 
### F-series v2 (Compute Optimized)
 
```
Standard_F2s_v2   ->  2 network interfaces
Standard_F4s_v2   ->  2 network interfaces
Standard_F8s_v2   ->  4 network interfaces
Standard_F16s_v2  ->  4 network interfaces
Standard_F32s_v2  ->  8 network interfaces
Standard_F48s_v2  ->  8 network interfaces
Standard_F64s_v2  ->  8 network interfaces
Standard_F72s_v2  ->  8 network interfaces
```
 
### Quick Rule of Thumb
 
| vCPU Count | Typical Max NICs |
|------------|-----------------|
| 1-2 vCPUs  | 2 NICs          |
| 4 vCPUs    | 2-4 NICs        |
| 8 vCPUs    | 4 NICs          |
| 16+ vCPUs  | 8 NICs          |
 
> **Note:** Azure max is **8 NICs** for standard VMs.
>
> For the current configuration (`Standard_D2s_v3`), the maximum supported NICs is **2**.
 
## Azure Regions - Supported VM Families
 
### Region Availability Ranking
 
**Tier 1 - Best Availability** (largest datacenter footprint, all VM families):
 
| Region Name | Region Code | D-series v3 | D-series v5 | E-series v5 | F-series v2 |
|-------------|-------------|-------------|-------------|-------------|-------------|
| East US | `eastus` | Y | Y | Y | Y |
| East US 2 | `eastus2` | Y | Y | Y | Y |
| West Europe | `westeurope` | Y | Y | Y | Y |
| West US 2 | `westus2` | Y | Y | Y | Y |
| South Central US | `southcentralus` | Y | Y | Y | Y |
| North Europe | `northeurope` | Y | Y | Y | Y |
 
**Tier 2 - Excellent Availability** (all major VM families supported):
 
| Region Name | Region Code | D-series v3 | D-series v5 | E-series v5 | F-series v2 |
|-------------|-------------|-------------|-------------|-------------|-------------|
| Central US | `centralus` | Y | Y | Y | Y |
| UK South | `uksouth` | Y | Y | Y | Y |
| France Central | `francecentral` | Y | Y | Y | Y |
| Southeast Asia | `southeastasia` | Y | Y | Y | Y |
| Japan East | `japaneast` | Y | Y | Y | Y |
| West US 3 | `westus3` | Y | Y | Y | Y |
| Korea Central | `koreacentral` | Y | Y | Y | Y |
| Germany West Central | `germanywestcentral` | Y | Y | Y | Y |
| Central India | `centralindia` | Y | Y | Y | Y |
 
**Tier 3 - Good Availability** (some newer VM series may be limited):
 
| Region Name | Region Code | D-series v3 | D-series v5 | E-series v5 | F-series v2 |
|-------------|-------------|-------------|-------------|-------------|-------------|
| North Central US | `northcentralus` | Y | Y | Y | Y |
| East Asia | `eastasia` | Y | Y | Y | Y |
| West US | `westus` | Y | Y | Y | Y |
| Japan West | `japanwest` | Y | Y | Y | Y |
| Brazil South | `brazilsouth` | Y | Y | Y | Y |
| Australia East | `australiaeast` | Y | Y | Y | Y |
| Canada Central | `canadacentral` | Y | Y | Y | Y |
| Norway East | `norwayeast` | Y | Y | Y | Y |
| Switzerland North | `switzerlandnorth` | Y | Y | Y | Y |
| UAE North | `uaenorth` | Y | Y | Y | Y |
| South Africa North | `southafricanorth` | Y | Y | Y | Y |
 
### Region + VM Size + Max NICs (Complete Reference)
 
**US Regions:**
 
```
East US (eastus)
  Standard_D2s_v3   ->  2 NICs    Standard_D2s_v5   ->  2 NICs    Standard_F2s_v2   ->  2 NICs
  Standard_D4s_v3   ->  2 NICs    Standard_D4s_v5   ->  2 NICs    Standard_F4s_v2   ->  2 NICs
  Standard_D8s_v3   ->  4 NICs    Standard_D8s_v5   ->  4 NICs    Standard_F8s_v2   ->  4 NICs
  Standard_D16s_v3  ->  8 NICs    Standard_D16s_v5  ->  8 NICs    Standard_F16s_v2  ->  4 NICs
  Standard_D32s_v3  ->  8 NICs    Standard_D32s_v5  ->  8 NICs    Standard_F32s_v2  ->  8 NICs
  Standard_D64s_v3  ->  8 NICs    Standard_D96s_v5  ->  8 NICs    Standard_F72s_v2  ->  8 NICs
 
East US 2 (eastus2)             -> Same VM sizes and NIC limits as East US
Central US (centralus)          -> Same VM sizes and NIC limits as East US
West US 2 (westus2)             -> Same VM sizes and NIC limits as East US
West US 3 (westus3)             -> Same VM sizes and NIC limits as East US
South Central US (southcentralus) -> Same VM sizes and NIC limits as East US
North Central US (northcentralus) -> Same VM sizes and NIC limits as East US
```
 
**Europe Regions:**
 
```
West Europe (westeurope)
  Standard_D2s_v3   ->  2 NICs    Standard_D2s_v5   ->  2 NICs    Standard_E2s_v5   ->  2 NICs
  Standard_D4s_v3   ->  2 NICs    Standard_D4s_v5   ->  2 NICs    Standard_E4s_v5   ->  2 NICs
  Standard_D8s_v3   ->  4 NICs    Standard_D8s_v5   ->  4 NICs    Standard_E8s_v5   ->  4 NICs
  Standard_D16s_v3  ->  8 NICs    Standard_D16s_v5  ->  8 NICs    Standard_E16s_v5  ->  8 NICs
  Standard_D32s_v3  ->  8 NICs    Standard_D32s_v5  ->  8 NICs    Standard_E32s_v5  ->  8 NICs
  Standard_D64s_v3  ->  8 NICs    Standard_D96s_v5  ->  8 NICs    Standard_E96s_v5  ->  8 NICs
 
North Europe (northeurope)             -> Same VM sizes and NIC limits as West Europe
UK South (uksouth)                     -> Same VM sizes and NIC limits as West Europe
France Central (francecentral)         -> Same VM sizes and NIC limits as West Europe
Germany West Central (germanywestcentral) -> Same VM sizes and NIC limits as West Europe
```
 
**Asia Regions:**
 
```
Southeast Asia (southeastasia)
  Standard_D2s_v3   ->  2 NICs    Standard_D2s_v5   ->  2 NICs    Standard_E2s_v5   ->  2 NICs
  Standard_D4s_v3   ->  2 NICs    Standard_D4s_v5   ->  2 NICs    Standard_E4s_v5   ->  2 NICs
  Standard_D8s_v3   ->  4 NICs    Standard_D8s_v5   ->  4 NICs    Standard_E8s_v5   ->  4 NICs
  Standard_D16s_v3  ->  8 NICs    Standard_D16s_v5  ->  8 NICs    Standard_E16s_v5  ->  8 NICs
  Standard_D32s_v3  ->  8 NICs    Standard_D32s_v5  ->  8 NICs    Standard_E32s_v5  ->  8 NICs
  Standard_D64s_v3  ->  8 NICs    Standard_D96s_v5  ->  8 NICs    Standard_E96s_v5  ->  8 NICs
 
East Asia (eastasia)           -> Same VM sizes and NIC limits as Southeast Asia
Japan East (japaneast)         -> Same VM sizes and NIC limits as Southeast Asia
Korea Central (koreacentral)   -> Same VM sizes and NIC limits as Southeast Asia
Central India (centralindia)   -> Same VM sizes and NIC limits as Southeast Asia
```
 
### How to Check VM Availability in a Region
 
```bash
# List all available VM SKUs in a specific region
az vm list-skus --location eastus --output table
 
# Filter by VM family
az vm list-skus --location centralus --size Standard_D --output table
az vm list-skus --location westeurope --size Standard_E --output table
```
 
## Security Notes
 
- `terraform.tfvars` contains sensitive credentials and is excluded from git via `.gitignore`
- The `client_secret` variable is marked as `sensitive` in Terraform
- The NSG allows SSH (port 22) and HTTP (port 80) from all sources (`0.0.0.0/0`). For production, restrict `source_address_prefix` to your IP
- Never commit `.tfstate` files as they may contain sensitive data
