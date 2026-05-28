---
lab:
  title: 'Lab: Integrated Networking, Compute, and Storage'
  module: 'Azure Bootcamp Intermediate'
---

# Lab – Integrated Networking, Compute, and Storage

## Lab Introduction

In this lab you build a complete multi-tier Azure architecture that integrates networking, compute, and storage services. You will work through the following tasks:
- Design and create virtual networks with segmented subnets
- Protect subnets with Network Security Groups
- Secure VM access with Azure Bastion
- Provide name resolution with public and private Azure DNS zones
- Deploy virtual machines using both portal and CLI approaches
- Configure virtual network peering to connect segmented networks
- Implement storage solutions with Blob lifecycle management and Azure Files
- Deploy infrastructure using ARM templates and Bicep
- Distribute traffic at Layer 4 using Azure Load Balancer
- Route traffic at Layer 7 using Azure Application Gateway with path-based routing

## Pre-requisites

- An Azure subscription
- Owner rights at the resource group level

## Estimated Timing: 165 minutes

## Lab Scenario

Your organisation is deploying a multi-tier retail application to Azure. The application has several distinct requirements that span networking, compute, and storage:

- Core IT services (database and other shared services) must live in a dedicated `CoreServicesVnet`
- The public-facing application tier runs in a separate `AppVnet` with two web server VMs
- The two VNets must be connected via peering while remaining isolated from the internet
- Secure VM access must be provided without exposing RDP ports directly to the internet
- Product images and order receipts need cost-effective Blob Storage with automatic tiering
- Web servers require a shared network file system (Azure Files) accessible from multiple VMs simultaneously
- Web traffic must be distributed across backend VMs using both Layer 4 and Layer 7 load balancing

You have been tasked with building this complete environment using a combination of portal, CLI, and Infrastructure as Code approaches.

## Architecture Overview

```
RG-Lab-Integrated-yourname
├── CoreServicesVnet (10.20.0.0/16)
│   ├── SharedServicesSubnet (10.20.10.0/24) → CoreServicesVM (Windows, Portal)
│   ├── DatabaseSubnet (10.20.20.0/24) → NSG applied
│   └── AzureBastionSubnet (10.20.30.0/26) → Azure Bastion
│
├── AppVnet (10.60.0.0/16)
│   ├── BackendSubnet1 (10.60.1.0/24) → vm0 (Windows + IIS, CLI)
│   ├── BackendSubnet2 (10.60.2.0/24) → vm1 (Windows + IIS, CLI)
│   └── AppGwSubnet (10.60.3.224/27) → Application Gateway
│
├── VNet Peering: CoreServicesVnet ↔ AppVnet
│   (allows app tier VMs to reach DatabaseSubnet)
│
├── Azure Bastion (Basic tier) → Secure RDP/SSH access to all VMs
│   └── Public IP: az104-bastion-pip
│
├── Network Security Group: myNSGSecure (→ DatabaseSubnet)
│   └── Inbound: allow TCP 1433 from AppVnet (10.60.0.0/16)
│
├── Public DNS Zone: adventuretravel.com
├── Private DNS Zone: private.adventuretravel.com (→ CoreServicesVnet)
│
├── Storage Account
│   ├── Blob Container: product-images (lifecycle: Hot→Cool→Archive)
│   └── File Share: erp-share (mounted as Z: on vm0, vm1)
│
├── Azure Load Balancer (Standard, Public) → vm0, vm1
└── Application Gateway (Standard V2) → /image/* → vm0, /video/* → vm1
```

## Job Skills

- Task 1: Create resource group and virtual networks
- Task 2: Protect the database subnet with a Network Security Group and setup Bastion for VM access
- Task 3: Configure public and private Azure DNS zones
- Task 4: Deploy CoreServicesVM via Portal
- Task 5: Deploy vm0 and vm1 via CLI with IIS
- Task 6: Deploy storage account using ARM and Bicep templates
- Task 7: Configure Blob Storage with lifecycle management
- Task 8: Create Azure Files and mount on both VMs
- Task 9: Configure virtual network peering
- Task 10: Configure Azure Load Balancer (Layer 4)
- Task 11: Configure Application Gateway with path-based routing (Layer 7)
- Task 12: Verification and cleanup

---

## Task 1: Create Resource Group and Virtual Networks

Virtual networks are the foundational building block for private networking in Azure. In this task you create the resource group and both VNets upfront — `CoreServicesVnet` for the internal tier and `AppVnet` for the application tier.

> **Design note:** Avoid overlapping IP address ranges across all virtual networks and any connected on-premises networks. Overlapping ranges block peering and increase troubleshooting complexity. Plan your IP addressing scheme across the entire environment before deploying.

### Create the resource group

1. Sign in to the [Azure portal](https://portal.azure.com).

2. Search for and select **Resource groups**, then select **+ Create**.

3. Configure the resource group:

   | Setting | Value |
   | --- | --- |
   | Subscription | Your subscription |
   | Resource group | `RG-Lab-Integrated-yourname` (replace `yourname` with your initials) |
   | Region | **Australia East** (keep consistent throughout the lab) |
   
   > **Note**: For EMEA-based students, use **West Europe**. The region must be the same for all resources in this lab.

4. Select **Review + create**, then **Create**.

### Create the CoreServicesVnet

5. Search for and select **Virtual Networks**, then select **+ Create**.

6. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Name | `CoreServicesVnet` |
   | Region | **Australia East** |

7. Select the **Address space** tab. Replace the default IPv4 address space with `10.20.0.0/16`.

8. Delete the default subnet, then select **+ Add a subnet** for each of the following. Select **Add** after each:

   | Subnet | Subnet name | Starting address | Size |
   | --- | --- | --- | --- |
   | First | `SharedServicesSubnet` | `10.20.10.0` | `/24` |
   | Second | `DatabaseSubnet` | `10.20.20.0` | `/24` |
   
   For the third subnet, select Azure Bastion as the Subnet purpose, then set the Starting address to `10.20.30.0` and size to `/26`:

   > **Note:** Every virtual network must have at least one subnet. Azure reserves five IP addresses within each subnet (the network address, broadcast address, and three addresses reserved by Azure), so a /24 gives you 251 usable addresses.
   
   > **Important:** The subnet named `AzureBastionSubnet` is required for Azure Bastion deployment. The name is case-sensitive and cannot be changed. The minimum subnet size is /26 (64 addresses), which provides enough IP addresses for Bastion instances and Azure's reserved addresses.

9. Select **Review + create**, then **Create**. Wait for the deployment to succeed.

### Create the AppVnet

10. Search for and select **Virtual Networks**. Select **+ Create** again.

11. On the **Basics** tab:

    | Setting | Value |
    | --- | --- |
    | Resource group | **RG-Lab-Integrated-yourname** |
    | Name | `AppVnet` |
    | Region | **Australia East** |

12. On the **Address space** tab, replace the address space with `10.60.0.0/16`.

13. Delete the default subnet and add the following subnets:

    | Subnet | Subnet name | Starting address | Size |
    | --- | --- | --- | --- |
    | First | `BackendSubnet1` | `10.60.1.0` | `/24` |
    | Second | `BackendSubnet2` | `10.60.2.0` | `/24` |
    | Third | `AppGwSubnet` | `10.60.3.224` | `/27` |

14. Select **Review + create**, then **Create**. Wait for the deployment to succeed.

15. Confirm both VNets appear in the portal. Navigate to each and verify the address space and subnets are correct.

---

## Task 2: Protect the Database Subnet with a Network Security Group

Network Security Groups (NSGs) filter inbound and outbound traffic using priority-ordered rules based on source, destination, protocol, and port. In this task you attach an NSG directly to the `DatabaseSubnet` so that only traffic from the `AppVnet` address space (`10.60.0.0/16`) can reach it on port 1433. All other inbound traffic is denied by the default rules, and outbound internet access is explicitly blocked.

### Create the Network Security Group

1. Search for and select **Network security groups**, then select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Name | `myNSGSecure` |
   | Region | **Australia East** |

3. Select **Review + create**, then **Create**.

4. Once deployed, select **Go to resource**.

### Add an inbound security rule

5. In the **Settings** blade, select **Inbound security rules**.

6. Select **+ Add** and configure:

   | Setting | Value |
   | --- | --- |
   | Source | **IP Addresses** |
   | Source IP addresses/CIDR ranges | `10.60.0.0/16` |
   | Source port ranges | `*` |
   | Destination | **Any** |
   | Service | **MS SQL** |
   | Destination port ranges | `1433` |
   | Protocol | **TCP** |
   | Action | **Allow** |
   | Priority | `100` |
   | Name | `AllowAppVnetToDatabase` |

7. Select **Add**.

### Add an outbound security rule to block internet

8. In the **Settings** blade, select **Outbound security rules**.

9. Select **+ Add** and configure:

   | Setting | Value |
   | --- | --- |
   | Source | **Any** |
   | Source port ranges | `*` |
   | Destination | **Service Tag** |
   | Destination service tag | **Internet** |
   | Service | **Custom** |
   | Destination port ranges | `*` |
   | Protocol | **Any** |
   | Action | **Deny** |
   | Priority | `100` |
   | Name | `DenyInternetOutbound` |

10. Select **Add**.

### Associate the NSG with the DatabaseSubnet

11. In the **Settings** blade, select **Subnets**.

12. Select **+ Associate**.

13. Configure:

    | Setting | Value |
    | --- | --- |
    | Virtual network | **CoreServicesVnet** |
    | Subnet | **DatabaseSubnet** |

14. Select **OK**.

15. Confirm the **Subnets** blade now shows **DatabaseSubnet**.

### Setup Bastion for VM access

Azure Bastion is a fully managed PaaS service that provides secure and seamless RDP and SSH connectivity to your virtual machines directly through the Azure portal over TLS. When you connect via Azure Bastion, your VMs do not need a public IP address, agent, or special client software. In this task you deploy Azure Bastion in the CoreServicesVnet to provide secure access to all VMs without exposing them to the public internet.

> **Security benefit:** Azure Bastion protects VMs from port scanning and other threats by eliminating the need to expose RDP (port 3389) or SSH (port 22) to the internet. All connections are established over HTTPS (port 443) from the Azure portal, providing an additional layer of security and audit capability.

### Create the Bastion Host

16. Search for and select **Bastions**, then select **+ Create**.

17. On the **Basics** tab, configure:

    | Setting | Value |
    | --- | --- |
    | Resource group | **RG-Lab-Integrated-yourname** |
    | Name | `az104-bastion` |
    | Region | **Australia East** |
    | Tier | **Basic** |
    | Instance count | `2` (default) |
    | Virtual network | **CoreServicesVnet** |
    | Subnet | **AzureBastionSubnet (10.20.30.0/26)** (auto-populated) |

18. Under **Public IP address**, configure:

    | Setting | Value |
    | --- | --- |
    | Public IP address | **Create new** |
    | Public IP address name | `az104-bastion-pip` |

19. Select **Review + create**, then **Create**.

    > **Note:** Azure Bastion deployment typically takes 8-10 minutes. You can continue to the next task while Bastion deploys. You will use Bastion to connect to CoreServicesVM in Task 4.

20. Wait for the deployment to complete. Once finished, select **Go to resource**.

21. On the Bastion **Overview** page, confirm the following:
    - **Virtual network** shows **CoreServicesVnet**
    - **Subnet** shows **AzureBastionSubnet**
    - **Public IP address** shows **az104-bastion-pip**

**Key point:** NSG rules are stateful — if an inbound connection is allowed, the corresponding return traffic is automatically permitted without requiring an explicit outbound rule. The default NSG rules allow inbound traffic from the same VNet and outbound traffic to the internet. The custom rules you added override these defaults for the DatabaseSubnet.

**Why Azure Bastion?** Bastion provides several security advantages: (1) No public IP addresses needed on VMs, (2) Protection against port scanning and zero-day exploits, (3) Centralized access control and audit logging, (4) TLS encryption for all connections, (5) No need to manage NSG rules for RDP/SSH access. Once deployed, Bastion can reach VMs in any peered VNet, making it ideal for hub-and-spoke network topologies.

---

## Task 3: Configure Public and Private Azure DNS Zones

Azure DNS provides both public and private DNS hosting. In this task you create a public DNS zone for internet-resolvable records and a private DNS zone for internal name resolution within and across your virtual networks.

### Create a public DNS zone

1. Search for and select **DNS zones**, then select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Name | `adventuretravel.com` |

   > **Note:** You do not need to own this domain for lab purposes. In production, you would configure the domain registrar to point to Azure's name servers.

3. Select **Review + create**, then **Create**.

4. Once deployed, select **Go to resource**.

5. Go to **DNS Management > Record sets** and select **+ Add** to add an A record:

   | Setting | Value |
   | --- | --- |
   | Name | `www` |
   | Type | **A** |
   | TTL | `1` |
   | TTL unit | **Hours** |
   | IP address | `10.60.1.4` (example IP — use any valid IP) |

6. Select **Add**.

### Create a private DNS zone

7. Search for and select **Private DNS zones**, then select **+ Create**.

8. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Name | `private.adventuretravel.com` |

9. Select **Review + create**, then **Create**.

10. Once deployed, select **Go to resource**.

### Link the private zone to CoreServicesVnet

11. In the **Settings** blade, select **Virtual network links**.

12. Select **+ Add** and configure:

    | Setting | Value |
    | --- | --- |
    | Link name | `CoreServicesVnetLink` |
    | Virtual network | **CoreServicesVnet** |
    | Enable auto registration | **Checked** |

    > **Auto-registration:** When enabled, Azure automatically creates and maintains DNS A records for every VM in the linked VNet. When a VM starts, its record is created. When it stops, the record is removed. No manual DNS management required.

13. Select **Create** and wait for the link to be created.

### Link the private zone to AppVnet (resolution only)

14. Select **+ Add** again and configure:

    | Setting | Value |
    | --- | --- |
    | Link name | `AppVnetLink` |
    | Virtual network | **AppVnet** |
    | Enable auto registration | **Unchecked** |

    > **Why no auto-registration?** The AppVnet VMs do not need their names registered in the private zone — they only need to resolve names registered from CoreServicesVnet. This is a resolution-only link.

15. Select **Create**.

16. Confirm both VNet links appear in the **Virtual network links** blade.

**Why private DNS?** VMs in both peered VNets can now resolve `<vmname>.private.adventuretravel.com` without the record being visible on the internet. When the VM is replaced or its IP changes, the auto-registered record updates automatically.

---

## Task 4: Deploy CoreServicesVM via Portal

In this task you deploy a Windows Server virtual machine using the Azure portal. This VM will be used to verify connectivity across the peered VNets. It will be deployed in the `SharedServicesSubnet` of the `CoreServicesVnet` and accessed securely via Azure Bastion without exposing RDP ports to the internet. It will be given an internal DNS name of `coreservicesvm.private.adventuretravel.com` via the private DNS zone. Our backend VMs in the AppVnet will be able to reach this VM on its private IP address once peering is configured in Task 9.

### Create the virtual machine

1. Search for and select **Virtual machines**, then select **+ Create → Azure virtual machine**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Virtual machine name | `CoreServicesVM` |
   | Region | **Australia East** |
   | Availability options | **No infrastructure redundancy required** |
   | Security type | **Standard** |
   | Image | **Windows Server 2022 Datacenter: Azure Edition Hotpatch – x64 Gen2** |
   | VM architecture | **x64** |
   | Size | **Standard_B2s** (2 vCPUs, 4 GiB RAM) |

3. Under **Administrator account**:

   | Setting | Value |
   | --- | --- |
   | Username | `azureuser` |
   | Password | A strong password (note this down) |
   | Confirm password | Re-enter the password |

4. Under **Inbound port rules**:

   | Setting | Value |
   | --- | --- |
   | Public inbound ports | **None** |

   > **Note:** We are using Azure Bastion (deployed in Task 2) for secure VM access. No public IP address or RDP port exposure is required — Bastion provides secure, browser-based RDP access over TLS.

5. Select the **Networking** tab:

   | Setting | Value |
   | --- | --- |
   | Virtual network | **CoreServicesVnet** |
   | Subnet | **SharedServicesSubnet (10.20.10.0/24)** |
   | Public IP | **None** |
   | NIC network security group | **Basic** |
   | Public inbound ports | **None** |

6. Select **Review + create**, then **Create**.

7. Wait for the deployment to complete (typically 2–3 minutes).

### Connect to the VM via Bastion

8. Once deployed, select **Go to resource**.

9. In the Overview blade, confirm the following:
    - **Virtual network** shows **CoreServicesVnet**
    - **Subnet** shows **SharedServicesSubnet**
    - **Public IP address** shows **None**
    - **Private IP address** shows `10.20.10.4`

10. Select **Connect** from the top toolbar, then select **Connect via Bastion** (or select the **Bastion** tab).

11. On the Bastion connection page, enter the credentials:

    | Setting | Value |
    | --- | --- |
    | Username | `azureuser` |
    | Password | The password you set during VM creation |

12. Select **Connect**. A new browser tab will open with the VM desktop session.

    > **How it works:** Azure Bastion establishes an RDP connection to the VM over its private IP address. The connection from your browser to Bastion uses HTTPS (port 443), and Bastion connects to the VM using RDP over the private network. No public IP address or exposed RDP port is required on the VM.

**Key point:** CoreServicesVM is now running in the CoreServicesVnet and can be accessed via Bastion securely. Once VNet peering is configured in Task 9, this VM will be reached by the backend VMs in the AppVnet using its private IP address.

---

## Task 5: Deploy vm0 and vm1 via CLI with IIS

In this task you deploy two Windows Server VMs in the AppVnet using Azure CLI. Each VM will have IIS installed automatically via a custom-data PowerShell script, and each will serve different content on `/image` and `/video` paths for the Application Gateway demonstration in Task 11.

### Create the NSG for backend VMs

1. Open **Cloud Shell** (Bash) from the top-right of the Azure portal. If prompted, select **Bash**. No need to create storage.

2. Verify your subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

3. If needed, set the correct subscription:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

4. Create the NSG for the backend VMs:

   ```bash
   az network nsg create \
     --resource-group RG-Lab-Integrated-yourname \
     --name app-nsg \
     --location australiaeast
   ```
   > Ensure you modify the Resource Group to match the one you created in Task 1.
   > For EMEA-based students, change the location to `westeurope`.

5. Add a rule to allow HTTP traffic:

   ```bash
   az network nsg rule create \
     --resource-group RG-Lab-Integrated-yourname \
     --nsg-name app-nsg \
     --name AllowHTTP \
     --priority 100 \
     --protocol Tcp \
     --destination-port-ranges 80 \
     --access Allow
   ```
   > Ensure you modify the Resource Group to match the one you created in Task 1.
   
### Deploy vm0 in BackendSubnet1

6. Create vm0:

   > **Important:** Replace `<password>` with a strong password of your choice. Use the same password for both VMs for simplicity. Make sure you keep the quotes around the password.
   > Ensure you modify the Resource Group to match the one you created in Task 1.

   ```bash
   az vm create \
   --resource-group RG-Lab-Integrated-yourname \
   --name az104-06-vm0 \
   --image Win2022Datacenter \
   --vnet-name AppVnet \
   --subnet BackendSubnet1 \
   --nsg app-nsg \
   --public-ip-address "" \
   --admin-username azureuser \
   --admin-password 'P@ssw0rd1234!ChangeMe' \
   --size Standard_B2s
   ```
   
   Install IIS and create custom pages:

   ```bash
   az vm run-command invoke \
   --resource-group RG-Lab-Integrated-yourname \
   --name az104-06-vm0 \
   --command-id RunPowerShellScript \
   --scripts @- <<'PS'
   Install-WindowsFeature -Name Web-Server -IncludeManagementTools

   Remove-Item C:\inetpub\wwwroot\iisstart.htm -ErrorAction SilentlyContinue
   Set-Content -Path "C:\inetpub\wwwroot\iisstart.htm" -Value "<h1>Hello World from az104-06-vm0</h1>"

   New-Item -Path "C:\inetpub\wwwroot\image" -ItemType Directory -Force
   Set-Content -Path "C:\inetpub\wwwroot\image\index.html" -Value "<h1>Image server - vm0</h1>"

   New-Item -Path "C:\inetpub\wwwroot\video" -ItemType Directory -Force
   Set-Content -Path "C:\inetpub\wwwroot\video\index.html" -Value "<h1>Video server - vm0</h1>"

   # Ensure HTTP allowed in Windows firewall
   netsh advfirewall firewall add rule name="Allow HTTP 80" dir=in action=allow protocol=TCP localport=80
   PS
   ```
   > This script installs IIS, removes the default page, and creates custom pages for the root, `/image`, and `/video` paths.
   

7. Wait for the deployment to complete (approximately 3–5 minutes).

### Deploy vm1 in BackendSubnet2

8. Create vm1:

   > **Important:** Replace `<password>` with a strong password of your choice. Use the same password for both VMs for simplicity. Make sure you keep the quotes around the password.
   > Ensure you modify the Resource Group to match the one you created in Task 1.

   ```bash
   az vm create \
   --resource-group RG-Lab-Integrated-yourname \
   --name az104-06-vm1 \
   --image Win2022Datacenter \
   --vnet-name AppVnet \
   --subnet BackendSubnet2 \
   --nsg app-nsg \
   --public-ip-address "" \
   --admin-username azureuser \
   --admin-password 'P@ssw0rd1234!ChangeMe' \
   --size Standard_B2s
   ```
   
   Install IIS and create custom pages:

   ```bash
   az vm run-command invoke \
   --resource-group RG-Lab-Integrated-yourname \
   --name az104-06-vm1 \
   --command-id RunPowerShellScript \
   --scripts @- <<'PS'
   Install-WindowsFeature -Name Web-Server -IncludeManagementTools

   Remove-Item C:\inetpub\wwwroot\iisstart.htm -ErrorAction SilentlyContinue
   Set-Content -Path "C:\inetpub\wwwroot\iisstart.htm" -Value "<h1>Hello World from az104-06-vm1</h1>"

   New-Item -Path "C:\inetpub\wwwroot\image" -ItemType Directory -Force
   Set-Content -Path "C:\inetpub\wwwroot\image\index.html" -Value "<h1>Image server - vm1</h1>"

   New-Item -Path "C:\inetpub\wwwroot\video" -ItemType Directory -Force
   Set-Content -Path "C:\inetpub\wwwroot\video\index.html" -Value "<h1>Video server - vm1</h1>"

   # Ensure HTTP allowed in Windows firewall
   netsh advfirewall firewall add rule name="Allow HTTP 80" dir=in action=allow protocol=TCP localport=80
   PS
   ```



9. Wait for the deployment to complete.

### Verify both VMs are running

10. List all VMs and their power states:

    > **Important:** Ensure you modify the Resource Group to match the one you created in Task 1.

    ```bash
    az vm list \
      --resource-group RG-Lab-Integrated-yourname \
      --show-details \
      --query "[].{Name:name, State:powerState, PrivateIP:privateIps}" \
      -o table
    ```

11. Confirm all three VMs show `VM running`:
    - CoreServicesVM
    - az104-06-vm0
    - az104-06-vm1

12. Note the private IP addresses. You will use these later.

**Key point:** The custom-data script runs during VM provisioning and installs IIS with custom HTML pages. This approach demonstrates Infrastructure as Code principles — the VM configuration is defined in the deployment script, not applied manually after deployment.

---

## Task 6: Deploy Storage Account Using ARM and Bicep Templates

In this task you deploy a storage account using an ARM template that demonstrates all five template sections: parameters, variables, functions, resources, and outputs. You will then convert the ARM template to Bicep, deploy it, and compare the two approaches. This demonstrates Infrastructure as Code best practices for repeatable, version-controlled deployments.

In our lab scenario, the Storage Account can be used to store product images and order receipts for the retail application. 

### Create the ARM template

1. In Cloud Shell, create a new file called `storage-template.json`:

   ```bash
   code storage-template.json
   ```

2. Paste the following ARM template:

   ```json
   {
       "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
       "contentVersion": "1.0.0.0",
   
       "parameters": {
           "storageAccountName": {
               "type": "string",
               "metadata": {
                   "description": "Base name for the storage account."
               }
           },
           "environment": {
               "type": "string",
               "defaultValue": "dev",
               "allowedValues": [ "dev", "test", "prod" ],
               "metadata": {
                   "description": "Deployment environment."
               }
           },
           "location": {
               "type": "string",
               "defaultValue": "[resourceGroup().location]",
               "metadata": {
                   "description": "Location for all resources."
               }
           }
       },
   
       "variables": {
           "storageSku": "[if(equals(parameters('environment'), 'prod'), 'Standard_GRS', 'Standard_LRS')]",
           "fullStorageName": "[concat(parameters('storageAccountName'), parameters('environment'))]"
       },
   
       "functions": [
           {
               "namespace": "myOrg",
               "members": {
                   "uniqueStorageName": {
                       "parameters": [
                           {
                               "name": "baseName",
                               "type": "string"
                           }
                       ],
                       "output": {
                           "type": "string",
                           "value": "[toLower(concat(parameters('baseName'), uniqueString(resourceGroup().id)))]"
                       }
                   }
               }
           }
       ],
   
       "resources": [
           {
               "type": "Microsoft.Storage/storageAccounts",
               "apiVersion": "2023-01-01",
               "name": "[myOrg.uniqueStorageName(variables('fullStorageName'))]",
               "location": "[parameters('location')]",
               "sku": {
                   "name": "[variables('storageSku')]"
               },
               "kind": "StorageV2",
               "properties": {
                   "accessTier": "Hot",
                   "minimumTlsVersion": "TLS1_2",
                   "supportsHttpsTrafficOnly": true
               }
           }
       ],
   
       "outputs": {
           "storageAccountName": {
               "type": "string",
               "value": "[myOrg.uniqueStorageName(variables('fullStorageName'))]"
           },
           "storageAccountId": {
               "type": "string",
               "value": "[resourceId('Microsoft.Storage/storageAccounts', myOrg.uniqueStorageName(variables('fullStorageName')))]"
           },
           "blobEndpoint": {
               "type": "string",
               "value": "[reference(myOrg.uniqueStorageName(variables('fullStorageName'))).primaryEndpoints.blob]"
           }
       }
   }
   ```

3. Save the file: **Ctrl+S**, then close the editor: **Ctrl+Q**.

### Understand the template structure

| Section | Purpose |
| --- | --- |
| **parameters** | Accept `storageAccountName` and `environment` at deploy time with validation |
| **variables** | Compute the SKU (GRS for prod, LRS otherwise) and combine name + environment |
| **functions** | `myOrg.uniqueStorageName()` appends a deterministic hash for global uniqueness |
| **resources** | Deploys a `StorageV2` storage account using the computed values |
| **outputs** | Returns the final storage account name, ID, and blob endpoint after deployment |

### Run a what-if check

4. Before deploying, preview the changes:

    > **Important:** Ensure you modify the Resource Group to match the one you created in Task 1.

   ```bash
   az deployment group what-if \
     --resource-group RG-Lab-Integrated-yourname \
     --template-file storage-template.json \
     --parameters storageAccountName=stlabint environment=dev
   ```

5. Review the output and confirm the storage account shows as **Create** (green `+`).

### Deploy the ARM template

6. Deploy the template:

    > **Important:** Ensure you modify the Resource Group to match the one you created in Task 1.

   ```bash
   az deployment group create \
     --resource-group RG-Lab-Integrated-yourname \
     --template-file storage-template.json \
     --parameters storageAccountName=stlabint environment=dev
   ```

7. Wait for the deployment to complete. Confirm `"provisioningState": "Succeeded"` in the output.

8. Note the outputs — the storage account name will include a unique suffix like `stlabintdevxyz123abc`.

### Convert the ARM template to Bicep

9. Use the Azure CLI to decompile the ARM template to Bicep:

   ```bash
   az bicep decompile --file storage-template.json
   ```

10. Open the generated Bicep file:

    ```bash
    code storage-template.bicep
    ```

11. Review the Bicep syntax. Notice:
    - No `$schema` or `contentVersion` boilerplate
    - Parameters declared with `param` keyword and types
    - Variables use `var` keyword
    - Resources use a clean `resource` block with symbolic names
    - Functions are expressed as Bicep functions
    - Cleaner, more readable syntax overall

12. Close the editor: **Ctrl+Q**.

### Deploy using Bicep

13. Modify the storage account name to avoid conflict:

    > **Important:** Ensure you modify the Resource Group to match the one you created in Task 1.

    ```bash
    az deployment group create \
      --resource-group RG-Lab-Integrated-yourname \
      --template-file storage-template.bicep \
      --parameters storageAccountName=stbicep environment=dev
    ```

14. Confirm `"provisioningState": "Succeeded"`.

15. List storage accounts to confirm both deployments exist:

    > **Important:** Ensure you modify the Resource Group to match the one you created in Task 1.

    ```bash
    az storage account list \
      --resource-group RG-Lab-Integrated-yourname \
      --query "[].name" \
      -o table
    ```

**Key point:** ARM templates are the native Azure Infrastructure as Code format. Bicep is a domain-specific language that compiles to ARM JSON and offers cleaner syntax, type safety, and better tooling. Both approaches enable version-controlled, repeatable deployments. Use Bicep for new projects.

---

## Task 7: Configure Virtual Network Peering

Virtual network peering creates a direct, low-latency connection between two VNets over the Microsoft backbone (not the public internet). In this task you configure bidirectional peering between CoreServicesVnet and AppVnet, allowing CoreServicesVM to reach the backend VMs in AppVnet.

### Configure peering from CoreServicesVnet to AppVnet

1. In the Azure portal, navigate to **CoreServicesVnet**.

2. Under **Settings**, select **Peerings**, then **+ Add**.

3. Fill in both sides of the peering simultaneously:

   | Parameter | Value |
   | --- | --- |
   | **Remote virtual network summary** | |
   | Peering link name | `AppVnet-to-CoreServicesVnet` |
   | Virtual network | `AppVnet` |
   | Allow 'AppVnet' to access 'CoreServicesVnet' | **Checked** (default) |
   | Allow 'AppVnet' to receive forwarded traffic from 'CoreServicesVnet' | **Checked** |
   | **Local virtual network summary** | |
   | Peering link name | `CoreServicesVnet-to-AppVnet` |
   | Allow 'CoreServicesVnet' to access 'AppVnet' | **Checked** (default) |
   | Allow 'CoreServicesVnet' to receive forwarded traffic from 'AppVnet' | **Checked** |

4. Select **Add** and wait a few seconds.

5. In the **Peerings** blade of `CoreServicesVnet`, confirm the **Peering status** shows **Connected**. Refresh if needed.

6. Navigate to **AppVnet → Peerings** and verify the reverse peering also shows **Connected**.

### Verify connectivity with Network Watcher

7. Search for and select **Network Watcher**.

8. In the **Network diagnostic tools** section, select **Connection troubleshoot**.

9. Configure the test:

   | Field | Value |
   | --- | --- |
   | Source type | **Virtual machine** |
   | Virtual machine | **CoreServicesVM** |
   | Destination type | **Virtual machine** |
   | Virtual machine | **az104-06-vm0** |
   | Protocol | **TCP** |
   | Destination port | `80` |

10. Select **Check**. Confirm the result shows **Reachable** — traffic from CoreServicesVnet is flowing to AppVnet over the peering.

### Verify cross-VNet private DNS resolution

Now verify that VMs in AppVnet can resolve DNS records auto-registered in CoreServicesVnet through the private DNS zone link. This demonstrates that the private DNS zone is working correctly across peered VNets.

11. From your CoreServicesVM Bastion session (or reconnect if needed), open **Remote Desktop Connection** to connect to vm0.

12. In **Computer**, enter vm0's private IP address (e.g., `10.60.1.4` — use the IP from Task 5, step 12).

13. When prompted, enter:
    - **Username:** `azureuser`
    - **Password:** the password you set when creating vm0

14. Once connected to vm0, open **PowerShell**.

15. Test DNS resolution from AppVnet to CoreServicesVnet:

    ```powershell
    nslookup coreservicesvm.private.adventuretravel.com
    ```

16. Confirm it resolves to CoreServicesVM's private IP address (should be 10.20.10.x).

    > **What this proves:** This confirms three things: (1) The AppVnet VNet link allows DNS resolution from the private DNS zone, (2) CoreServicesVM was auto-registered when it started, (3) DNS queries work across peered VNets. VMs in AppVnet can now discover and connect to services in CoreServicesVnet using friendly DNS names instead of memorizing IP addresses.

17. From the same PowerShell window on vm0, test connectivity to CoreServicesVM:

    ```powershell
    Test-NetConnection -ComputerName 10.60.1.4 -Port 80
    ```

    Replace `10.60.1.4` with vm0's actual private IP address (this tests connectivity back to the AppVnet from CoreServicesVM).

18. Disconnect from vm0 and return to the CoreServicesVM session.

**Key point:** Virtual network peering is non-transitive by default. If VNet A peers with VNet B, and VNet B peers with VNet C, VNet A cannot reach VNet C through VNet B. To achieve transitive routing, use either a Network Virtual Appliance or Azure Virtual WAN.

---

## Task 8: Configure Blob Storage with Lifecycle Management

In this task you create a blob container, upload files, and configure a lifecycle management policy to automatically move blobs between access tiers as they age. This reduces storage costs without application code changes.

### Create a blob container

1. In the Azure portal, navigate to one of the storage accounts created in Task 6 (e.g., `stlabintdevxyz123abc`).

2. In the left menu under **Data storage**, select **Containers**.

3. Select **+ Add container** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `product-images` |
   | Public access level | **Private (no anonymous access)** |

4. Select **Create**.

### Upload a blob and change its access tier

5. Open the `product-images` container. Select **Upload**.

6. Select any small file from your local machine (a JPG, PNG, or PDF works well).

7. Expand **Advanced** and review the **Access tier** dropdown — notice the options: Hot, Cool, and Cold. Leave it as **Hot (inferred)**.

8. Select **Upload**.

9. Once the upload completes, select the uploaded file to open its properties blade.

10. Review the **Access tier** field. Confirm it shows **Hot (inferred)** — the blob inherits the storage account's default tier.

11. Select **Change tier** at the top of the blade.

12. Change the tier to **Cool** and select **Save**.

13. Return to the blob list and confirm the **Access tier** column now shows **Cool**.

**Access tier comparison:**

| Tier | Storage cost | Retrieval cost | Minimum storage duration | Suitable for |
| --- | --- | --- | --- | --- |
| Hot | Highest | Lowest | None | Frequently accessed data |
| Cool | Medium | Medium | 30 days | Infrequently accessed, read within 30 days |
| Cold | Low | Higher | 90 days | Rarely accessed, read within 90 days |
| Archive | Lowest | Highest + rehydration delay | 180 days | Long-term retention, rare access |

### Configure a lifecycle management policy

14. In the storage account, under **Data management**, select **Lifecycle management**.

15. Select **+ Add a rule** and configure the **Details** tab:

    | Setting | Value |
    | --- | --- |
    | Rule name | `tier-to-cool-then-archive` |
    | Rule scope | **Apply rule to all blobs in your storage account** |
    | Blob type | **Block blobs** |
    | Blob subtype | **Base blobs** |

16. Select **Next** to open the **Base blobs** tab.

17. Configure the following transitions (select **+ Add condition** after each):

    | Condition | Days after last modification | Action |
    | --- | --- | --- |
    | Move to Cool storage | `30` | ✔ |
    | Move to Cold storage | `90` | ✔ |
    | Move to Archive storage | `180` | ✔ |
    | Delete the blob | `365` | ✔ |

18. Select **Add** to save the rule.

19. Return to the **Lifecycle management** blade and confirm the rule appears in the list.

**Why this matters:** Product images uploaded more than 30 days ago are unlikely to be accessed frequently. Automatically moving them to Cool and eventually Archive reduces storage cost without any application code changes. Lifecycle policies are evaluated once per day.

---

## Task 9: Create Azure Files and Mount on Both VMs

Azure Files provides fully managed SMB file shares hosted inside a standard Azure Storage account. Unlike Blob Storage, which exposes objects via HTTP/HTTPS, Azure Files exposes a true file system hierarchy that Windows machines can mount as a network drive (Z:).

### Create the Azure File Share

1. In the storage account, under **Data storage**, select **Classic file shares**.

2. Select **+ Classic file share** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `erp-share` |
   | Tier | **Transaction optimized** |

3. Select **Review + Create**, then **Create**.

4. Open the `erp-share` file share.

5. Select **+ Add directory** and create two folders:
   - `invoices`
   - `reports`

6. Under **Browse**, select the `invoices` directory, then **Upload** and upload any small file from your local machine as a test fixture.

### Retrieve the mount script for vm0

7. Navigate back to the `erp-share` overview page.

8. Select **Connect** from the top toolbar.

9. Select **Windows**, then **Show script**.

10. Copy the entire PowerShell script to your clipboard.

### Mount the share on vm0

11. Switch to your CoreServicesVM RDP session (from Task 4).

12. Open **Server Manager** → **Local Server**. Verify **IE Enhanced Security Configuration** is **Off** for Administrators. If not, turn it off to allow browsing to other VMs.

13. Open a browser inside the RDP session.

14. Navigate to the Azure portal and sign in.

15. Navigate to **Virtual machines** → **az104-06-vm0**.

16. Note the private IP address (e.g., `10.60.1.4`).

17. In the browser address bar, navigate to `http://10.60.1.4` (replace with vm0's actual private IP).

18. If you see "Hello World from az104-06-vm0", IIS is working correctly.

19. Now, to mount the Azure File Share on vm0, you need to RDP into vm0. From the Azure portal (inside the CoreServicesVM RDP session):

    > **Note:** While we could use Azure Bastion to connect directly to vm0 from the portal, connecting from CoreServicesVM via RDP over the private IP demonstrates an important concept: VNet peering allows private traffic to flow between networks. This simulates a real-world scenario where a jump box or bastion host in a management VNet connects to workload VMs in other VNets.

20. Go to **az104-06-vm0** → **Connect** → **Connect**.

21. Since vm0 has no public IP, you will connect via the private IP from CoreServicesVM. Open **Remote Desktop Connection** on CoreServicesVM.

22. In **Computer**, enter vm0's private IP address (e.g., `10.60.1.4`).

23. When prompted, enter:
    - **Username:** `azureuser`
    - **Password:** the password you set when creating vm0

24. Once connected to vm0, open **PowerShell** as Administrator.

25. Paste the mount script you copied earlier and press **Enter**.

26. Verify the mount:

    ```powershell
    Get-PSDrive Z
    ```

27. The output should show drive `Z:` with a `Root` of `\\storageaccountname.file.core.windows.net\erp-share`.

28. Open **File Explorer** and confirm drive `Z:` appears under **This PC**. Open it — you should see the `invoices` and `reports` directories.

### Mount the share on vm1

29. Disconnect from vm0's RDP session (back to CoreServicesVM).

30. Repeat steps 20-28 for **az104-06-vm1** using its private IP address.

### Verify shared storage

31. Once connected to vm1 via RDP, open **File Explorer** and navigate to `Z:\invoices`.

32. Create a new text file:

    ```powershell
    "Test from vm1" | Out-File Z:\invoices\test-from-vm1.txt
    ```

33. Open **File Explorer** on vm1 and confirm the file appears in `Z:\invoices`.

34. Switch back to vm0's RDP session (or reconnect if you disconnected).

35. Open **File Explorer** on vm0 and navigate to `Z:\invoices`.

36. Confirm `test-from-vm1.txt` appears immediately — this demonstrates that Azure Files behaves like a real network share with real-time synchronization.

**Key point:** Azure Files provides true SMB 3.0 file shares that multiple VMs can mount simultaneously. Unlike Blob Storage, which requires HTTP clients and has no concept of file locking, Azure Files supports standard Windows file operations, NTFS permissions (when using Active Directory integration), and concurrent access from multiple clients.

---


## Task 10: Configure Azure Load Balancer (Layer 4)

Azure Load Balancer operates at Layer 4 (TCP/UDP) and distributes inbound traffic across backend virtual machines. The Standard SKU provides zone redundancy, SLA guarantees, and health probes. In this task you create a public Load Balancer and add vm0 and vm1 to the backend pool.

### Create the Load Balancer

1. Search for and select **Load balancers**, then **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Name | `az104-lb` |
   | Region | **Australia East** |
   | SKU | **Standard** |
   | Type | **Public** |
   | Tier | **Regional** |

3. Select **Next: Frontend IP configuration**, then **+ Add a frontend IP configuration**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-fe` |
   | IP type | **IP address** |
   | Public IP address | Select **Create new** |

4. In the **Add a public IP address** dialog:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-lbpip` |
   | SKU | **Standard** |
   | Assignment | **Static** |
   | Availability zone | **Zone-redundant** |

5. Select **OK**, then **Add**.

6. Select **Next: Backend pools**, then **+ Add a backend pool**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-be` |
   | Virtual network | **AppVnet (RG-Lab-Integrated-yourname)** |
   | Backend Pool Configuration | **NIC** |
   | IP Version | **IPv4** |

7. Under **Virtual machines**, select **+ Add**:

8. Select both `az104-06-vm0` and `az104-06-vm1`. Select **Add**.

9. Select **Save**.

10. Select **Review + create**, then **Create**. Wait for deployment.

### Add a health probe

11. Once deployed, select **Go to resource**.

12. Under **Settings**, select **Health probes**, then **+ Add**:

    | Setting | Value |
    | --- | --- |
    | Name | `az104-hp` |
    | Protocol | **TCP** |
    | Port | `80` |
    | Interval | `5` |

13. Select **Save**.

### Add a load balancing rule

14. Under **Settings**, select **Load balancing rules**, then **+ Add**:

    | Setting | Value |
    | --- | --- |
    | Name | `az104-lbrule` |
    | IP Version | **IPv4** |
    | Frontend IP address | **az104-fe** |
    | Backend pool | **az104-be** |
    | Protocol | **TCP** |
    | Port | `80` |
    | Backend port | `80` |
    | Health probe | **az104-hp** |
    | Session persistence | **None** |
    | Idle timeout (minutes) | `4` |
    | TCP reset | **Disabled** |
    | Floating IP | **Disabled** |

15. Select **Save**.

### Test the Load Balancer

16. On the **Overview** blade of `az104-lb`, copy the **Public IP address** (listed under Frontend IP configuration).

17. Open a browser and navigate to `http://<frontend-ip-address>`.

18. You should see **Hello World from az104-06-vm0** or **az104-06-vm1**.

19. Refresh the page several times (or use an InPrivate/Incognito window). Confirm that responses alternate between both virtual machines — this demonstrates round-robin load distribution.

**Key point:** The Standard Load Balancer is zone-redundant and SLA-backed. Health probes monitor backend instance health — if a probe fails, the Load Balancer stops sending traffic to that instance until it recovers. Session persistence (sticky sessions) can be enabled if your application requires a client to always reach the same backend.

---

## Task 11: Configure Application Gateway with Path-Based Routing (Layer 7)

Azure Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer. Unlike the standard Load Balancer which routes based on IP and port, Application Gateway can route based on the URL path — directing `/image/*` traffic to vm0 and `/video/*` traffic to vm1.

### Create the Application Gateway

1. Search for and select **Application gateways**, then **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab-Integrated-yourname** |
   | Application gateway name | `az104-appgw` |
   | Region | **Australia East** |
   | Tier | **Standard V2** |
   | Enable autoscaling | **No** |
   | Instance count | `2` |
   | Availability zone | **None** |
   | HTTP2 | **Disabled** |
   | Virtual network | **AppVnet** |
   | Subnet | **AppGwSubnet (10.60.3.224/27)** |

3. Select **Next: Frontends**:

   | Setting | Value |
   | --- | --- |
   | Frontend IP address type | **Public** |
   | Public IP address | **Add new** → name `az104-gwpip` → **OK** |

4. Select **Next: Backends**.

5. Select **+ Add a backend pool**:

   **Default pool (both VMs):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-appgwbe` |
   | Add backend pool without targets | **No** |
   | Target type | **Virtual machine** |
   | Target | **az104-06-vm0 (NIC)** |

6. Select **Add** to add vm0, then select **+ Add target** and add:

   | Setting | Value |
   | --- | --- |
   | Target type | **Virtual machine** |
   | Target | **az104-06-vm1 (NIC)** |

7. Select **Add** to save the backend pool.

8. Select **+ Add a backend pool** again:

   **Images pool (vm0 only):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-imagebe` |
   | Target type | **Virtual machine** |
   | Target | **az104-06-vm0 (NIC)** |

9. Select **Add**.

10. Select **+ Add a backend pool** again:

    **Videos pool (vm1 only):**

    | Setting | Value |
    | --- | --- |
    | Name | `az104-videobe` |
    | Target type | **Virtual machine** |
    | Target | **az104-06-vm1 (NIC)** |

11. Select **Add**.

12. Select **Next: Configuration**, then **+ Add a routing rule**:

    | Setting | Value |
    | --- | --- |
    | Rule name | `az104-gwrule` |
    | Priority | `10` |

13. On the **Listener** tab:

    | Setting | Value |
    | --- | --- |
    | Listener name | `az104-listener` |
    | Frontend IP | **Public IPv4** |
    | Protocol | **HTTP** |
    | Port | `80` |
    | Listener type | **Basic** |
    | Error page url | **No** |

14. On the **Backend targets** tab:

    | Setting | Value |
    | --- | --- |
    | Target type | **Backend pool** |
    | Backend target | `az104-appgwbe` |
    | Backend settings | **Add new** |

15. In the **Add Backend setting** dialog:

    | Setting | Value |
    | --- | --- |
    | Backend settings name | `az104-http` |
    | Backend protocol | **HTTP** |
    | Backend port | `80` |
    | Cookie-based affinity | **Disable** |
    | Connection draining | **Disable** |
    | Request time-out (seconds) | `20` |

16. Select **Add** to save the backend setting.

17. Back on the **Backend targets** tab, select **Add multiple targets to create a path-based rule**.

18. Add the first path rule:

    **Image routing rule:**

    | Setting | Value |
    | --- | --- |
    | Path | `/image/*` |
    | Target name | `images` |
    | Backend settings | **az104-http** |
    | Backend target | `az104-imagebe` |

19. Select **Add**.

20. Add the second path rule:

    **Video routing rule:**

    | Setting | Value |
    | --- | --- |
    | Path | `/video/*` |
    | Target name | `videos` |
    | Backend settings | **az104-http** |
    | Backend target | `az104-videobe` |

21. Select **Add**.

22. Select **Add** to save the routing rule.

23. Select **Next: Tags**, then **Next: Review + create**, then **Create**.

    > **Note:** The Application Gateway takes 5–10 minutes to deploy. Continue to Task 12 while it deploys.

### Test path-based routing

24. Once deployed, navigate to **az104-appgw → Overview** and copy the **Frontend public IP address**.

25. Open a browser and navigate to `http://<frontend-ip>/image/`.

26. Confirm you see "Image server - vm0" — the Application Gateway routed the request to vm0 based on the `/image/*` path.

27. Open a new browser tab and navigate to `http://<frontend-ip>/video/`.

28. Confirm you see "Video server - vm1" — the request was routed to vm1 based on the `/video/*` path.

29. Navigate to `http://<frontend-ip>/` (no path). Confirm the default pool responds — either vm0 or vm1.

**What just happened:** The Application Gateway inspected the URL path of each request and forwarded it to the appropriate backend pool. This is Layer 7 routing: the gateway understands HTTP, not just TCP ports.

**Key point:** Application Gateway (Standard V2) supports SSL/TLS termination, WAF (upgraded to WAF V2 tier), cookie-based session affinity, connection draining, and custom health probes. Use Application Gateway when you need URL-based or host-header-based routing. Use Load Balancer when you only need TCP/UDP distribution.

---

## Task 12: Verification and Cleanup

### Verification checklist

1. **VNet Peering:** CoreServicesVM can reach vm0 and vm1 via private IPs
2. **Bastion Access:** Successfully connected to CoreServicesVM via Bastion without public IP
3. **DNS Resolution:** CoreServicesVM auto-registered in private DNS zone; vm0 can resolve coreservicesvm.private.adventuretravel.com to correct private IP
4. **Blob Lifecycle:** Lifecycle policy appears in storage account settings
5. **Azure Files:** Z: drive visible on vm0 and vm1, files shared in real-time
6. **Load Balancer:** Frontend IP alternates between vm0 and vm1 on refresh
7. **Application Gateway:** `/image/*` always routes to vm0, `/video/*` always routes to vm1

### Cleanup

**Remove all lab resources to avoid ongoing charges.**

1. In the Azure portal, navigate to **RG-Lab-Integrated-yourname**.

2. Select **Delete resource group**.

3. Type the resource group name to confirm, then select **Delete**.

4. Wait for the deletion to complete (approximately 5 minutes).

---

## Key Takeaways

- **Virtual networks and subnets** provide the foundation for all Azure networking. Avoid overlapping IP address ranges across VNets and on-premises networks from the outset.

- **NSGs** filter traffic at the subnet or NIC level using allow/deny rules evaluated in priority order. Attaching an NSG to a subnet enforces rules on all traffic entering or leaving that subnet.

- **Azure Bastion** provides secure, fully managed RDP/SSH access over TLS from the Azure portal without exposing VMs via public IP addresses or opening RDP ports to the internet. Bastion integrates with NSGs, requires a dedicated /26 or larger subnet named `AzureBastionSubnet`, and provides centralized access audit logs. Once deployed in a hub VNet, Bastion can reach VMs in any peered VNet.

- **Azure DNS** provides both public and private DNS hosting. Private zones are scoped to specific virtual networks. Enabling auto-registration on a VNet link means Azure automatically creates and maintains DNS records for every VM in that VNet.

- **Virtual network peering** connects two VNets directly over the Microsoft backbone with low latency. Peering is non-transitive — hub-and-spoke topologies require either a Network Virtual Appliance or Azure Virtual WAN for transitive routing.

- **Infrastructure as Code** (ARM templates and Bicep) enables version-controlled, repeatable deployments. Use what-if to preview changes before applying them. Use parameter files to manage environment-specific differences.

- **Blob lifecycle management** automates tier transitions and deletions based on blob age, reducing storage costs without application code changes.

- **Azure Files** provides true SMB file shares that multiple VMs can mount simultaneously as network drives, supporting standard Windows file operations and concurrent access.

- **Azure Load Balancer (Standard)** distributes TCP/UDP traffic at Layer 4 using health probes and configurable rules. Use it for VM-level traffic distribution where the application is not HTTP/HTTPS.

- **Azure Application Gateway** routes HTTP/HTTPS traffic at Layer 7 using URL path and host-header rules. Use it when requests must be directed to different backends based on content type, API version, or any other URL attribute.

---

## Cleanup

Ensure you have deleted the resource group **RG-Lab-Integrated-yourname** to avoid ongoing charges.
