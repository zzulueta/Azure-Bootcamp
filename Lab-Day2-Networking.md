---
lab:
  title: 'Lab Day 2: Networking'
  module: 'Advanced Azure Bootcamp – Day 2'
---

# Lab Day 2 – Networking

## Lab Introduction

In this lab you build a segmented network foundation in Azure and then progressively
connect and protect it. You will work through the following tasks:

- Design and create virtual networks with subnets
- Protect the database subnet with a Network Security Group
- Provide name resolution with public and private Azure DNS zones
- Connect segmented networks using virtual network peering
- Distribute traffic at Layer 4 using Azure Load Balancer
- Route traffic at Layer 7 using Azure Application Gateway with path-based rules

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab2**. 
Note: You will be assigned your own unique resource group name in the sandbox environment to avoid conflicts with other participants. Create a resource group in the same region you will deploy resources to (e.g., Australia East). Name the resource group RG-Lab2-yourname. Note the resource group name for use in all lab tasks.
- Owner rights at the resource group level

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation is deploying a two-tier Azure network environment. Core IT
services — DNS, security infrastructure, and a backend database — live in a dedicated
`CoreServicesVnet`. The public-facing application tier runs in a separate `AppVnet`
with two web server VMs. The two VNets must be connected so the application tier can
reach the database subnet, while remaining isolated from the internet.

You have been tasked with:

- Creating the `CoreServicesVnet` with subnets for shared services and the database tier
- Enforcing traffic controls using a Network Security Group on the database subnet
- Providing private and public DNS resolution
- Provisioning the `AppVnet` with backend VMs, then connecting it to `CoreServicesVnet` via peering
- Deploying and configuring an Azure Load Balancer for Layer 4 traffic distribution
- Deploying and configuring an Azure Application Gateway for Layer 7 path-based routing

## Architecture Overview

```
Subscription
└── RG-Lab2
    ├── CoreServicesVnet (10.20.0.0/16)
    │   ├── SharedServicesSubnet (10.20.10.0/24)
    │   └── DatabaseSubnet (10.20.20.0/24)  ← NSG applied
    │
    ├── AppVnet (10.60.0.0/16)
    │   ├── BackendSubnet1 (10.60.1.0/24)  ← vm0
    │   ├── BackendSubnet2 (10.60.2.0/24)  ← vm1
    │   └── AppGwSubnet (10.60.3.224/27)   ← Application Gateway
    │
    ├── VNet Peering: CoreServicesVnet ↔ AppVnet
    │   (allows app tier VMs to reach DatabaseSubnet)
    │
    ├── Network Security Group: myNSGSecure (→ DatabaseSubnet)
    │   └── Inbound: allow TCP 1433 from AppVnet (10.60.0.0/16)
    ├── Public DNS Zone: adventuretravel.com
    ├── Private DNS Zone: private.adventuretravel.com (→ CoreServicesVnet)
    ├── Azure Load Balancer (Standard, Public) → vm0, vm1
    └── Application Gateway (Standard V2, path-based) → /image/* vm0, /video/* vm1
```

## Job Skills

- Task 1: Create virtual networks and subnets
- Task 2: Protect the database subnet with a Network Security Group
- Task 3: Configure public and private Azure DNS zones
- Task 4: Deploy backend VMs and configure VNet peering
- Task 5: Configure an Azure Load Balancer
- Task 6: Configure an Azure Application Gateway with path-based routing

---

## Task 1: Create Virtual Networks and Subnets

Virtual networks are the foundational building block for private networking in Azure.
In this task you create both VNets upfront — `CoreServicesVnet` for the internal tier
and `AppVnet` for the application tier — so all subsequent tasks can reference them
without ordering dependencies.

> **Design note:** It is a good practice to avoid overlapping IP address ranges across all virtual networks and any connected on-premises networks. Overlapping ranges block peering and increase troubleshooting complexity. Plan your IP addressing scheme across the entire environment before deploying.

### Create the CoreServicesVnet

1. Sign in to the [Azure portal](https://portal.azure.com).

2. Search for and select **Virtual Networks**, then select **+ Create**.

3. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `CoreServicesVnet` |
   | Region | **Australia East** (or your preferred region — keep consistent throughout the lab) |

4. Select the **Address space** tab. Replace the default IPv4 address space with `10.20.0.0/16`.

5. Delete the default subnet, then select **+ Add a subnet** for each of the following. Select **Add** after each:

   | Subnet | Subnet name | Starting address | Size |
   | --- | --- | --- | --- |
   | First | `SharedServicesSubnet` | `10.20.10.0` | `/24` |
   | Second | `DatabaseSubnet` | `10.20.20.0` | `/24` |

   > **Note:** Every virtual network must have at least one subnet. Azure reserves five IP addresses within each subnet (the network address, broadcast address, and three addresses reserved by Azure), so a /24 gives you 251 usable addresses.

6. Select **Review + create**, then **Create**. Wait for the deployment to succeed.

### Create the AppVnet

7. Search for and select Virtual Networks. Select **+ Create** again.

8. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `AppVnet` |
   | Region | **Australia East** |

9. On the **Address space** tab, replace the address space with `10.60.0.0/16`.

10. Delete the default subnet and add the following subnets:

    | Subnet | Subnet name | Starting address | Size |
    | --- | --- | --- | --- |
    | First | `BackendSubnet1` | `10.60.1.0` | `/24` |
    | Second | `BackendSubnet2` | `10.60.2.0` | `/24` |

11. Select **Review + create**, then **Create**. Wait for the deployment to succeed.

12. Confirm both VNets appear in the portal. Navigate to each and verify the address space and subnets are correct.

---

## Task 2: Protect the Database Subnet with a Network Security Group

Network Security Groups (NSGs) filter inbound and outbound traffic using priority-ordered
rules based on source, destination, protocol, and port. In this task you attach an NSG
directly to the `DatabaseSubnet` so that only traffic from the `AppVnet` address space
(`10.60.0.0/16`) can reach it on port 1433. All other inbound traffic is denied by the
default rules, and outbound internet access is explicitly blocked.

### Create the Network Security Group

1. Search for and select **Network security groups**, then select **+ Create**.

2. Configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `myNSGSecure` |
   | Region | **Australia East** |

3. Select **Review + create**, then **Create**, then **Go to resource**.

### Associate the NSG with the database subnet

4. In the **Settings** blade of `myNSGSecure`, select **Subnets**, then **+ Associate**.

   | Setting | Value |
   | --- | --- |
   | Virtual network | **CoreServicesVnet (RG-Lab2)** |
   | Subnet | **DatabaseSubnet** |

5. Select **OK** to save the association.

### Add an inbound rule to allow app tier access

6. In the **Settings** blade, select **Inbound security rules**, then **+ Add**.

7. Configure the inbound rule:

   | Setting | Value |
   | --- | --- |
   | Source | **IP Addresses** |
   | Source IP addresses/CIDR ranges | `10.60.0.0/16` |
   | Source port ranges | `*` |
   | Destination | **Any** |
   | Service | **Custom** |
   | Destination port ranges | `1433` |
   | Protocol | **TCP** |
   | Action | **Allow** |
   | Priority | `100` |
   | Name | `AllowAppVnetToDatabase` |

8. Select **Add**.

### Add an outbound rule to deny internet access

9. Select **Outbound security rules**, then **+ Add**.

10. Configure the outbound rule:

    | Setting | Value |
    | --- | --- |
    | Source | **Any** |
    | Source port ranges | `*` |
    | Destination | **Service tag** |
    | Destination service tag | **Internet** |
    | Service | **Custom** |
    | Destination port ranges | `*` |
    | Protocol | **Any** |
    | Action | **Deny** |
    | Priority | `4096` |
    | Name | `DenyInternetOutbound` |

11. Select **Add**.

**Key point:** NSG rules are evaluated in priority order — lowest number is processed first. The default `AllowInternetOutBound` rule has priority 65001, so the `DenyInternetOutbound` rule at 4096 overrides it. By attaching the NSG directly to `DatabaseSubnet`, the rules apply to all traffic entering or leaving that subnet.

---

## Task 3: Configure Public and Private Azure DNS Zones

Azure DNS hosts DNS zones and provides name resolution using Azure's global
infrastructure. Public DNS zones serve internet-facing name resolution. Private DNS
zones provide name resolution exclusively within virtual networks — no public exposure.

### Configure a public DNS zone

1. Search for and select **DNS zones**, then select **+ Create**.

2. Configure:

   | Property | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `adventuretravel.com` |
   | Region | **Australia East** |

3. Select **Review + create**, then **Create**. Select **Go to resource**.

4. On the **Overview** blade, note the four Azure DNS name server addresses assigned to the zone — these would be registered with your domain registrar to delegate resolution to Azure.

5. In the **DNS Management** blade, select **Recordsets**, then **+ Add**.

   | Property | Value |
   | --- | --- |
   | Name | `www` |
   | Type | **A** |
   | TTL | `1` |
   | IP address | `10.20.10.4` |

   > **Note:** `10.20.10.4` is the first usable host address in `SharedServicesSubnet` — this is the private IP that `CoreServicesVM` will receive when deployed in Task 4. In a production scenario this record would point to a public IP or load balancer frontend.

6. Select **Add**. Confirm the `www` A record appears in the recordset list.

7. Open a local command prompt (Windows) or terminal (macOS/Linux) and run the following, replacing `<name server>` with one of the name server addresses from step 4:

   ```sh
   nslookup www.adventuretravel.com <name server>
   ```

   Verify the response resolves to `10.20.10.4`. This confirms the public DNS zone is working correctly.

### Configure a private DNS zone

8. Search for and select **Private DNS zones**, then select **+ Create**.

9. Configure:

   | Property | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Name | `private.adventuretravel.com` |

10. Select **Review + create**, then **Create**. Select **Go to resource**.

11. Notice the **Overview** blade shows no name server records — private zones are not registered in public DNS.

12. Link the zone to `CoreServicesVnet` with auto-registration enabled. In the **DNS Management** blade, select **Virtual network links**, then **+ Add**.

    | Property | Value |
    | --- | --- |
    | Link name | `coreservices-link` |
    | Virtual network | **CoreServicesVnet** |
    | Enable auto registration | **Checked** |

    Select **Create** and wait for the link status to show **Completed**.

13. Add a second link so `AppVnet` VMs can also resolve the private zone. Select **+ Add** again:

    | Property | Value |
    | --- | --- |
    | Link name | `appvnet-link` |
    | Virtual network | **AppVnet** |
    | Enable auto registration | Leave unchecked |

    Select **Create** and wait for the link status to show **Completed**.

    > **Note:** Auto-registration is only enabled for `CoreServicesVnet`. When `CoreServicesVM` is deployed in Task 4, Azure will automatically create an A record `coreservicesvm.private.adventuretravel.com` pointing to its private IP. `AppVnet` is linked as a resolver only — its VMs can look up the zone but won't have records auto-created.

**Why private DNS?** VMs in both peered VNets can now resolve `coreservicesvm.private.adventuretravel.com` without the record being visible on the internet. When the VM is replaced or its IP changes, the auto-registered record updates automatically — no manual DNS management required.

---

## Task 4: Deploy Backend VMs and Configure VNet Peering

In this task you deploy three VMs via Cloud Shell — `CoreServicesVM` in `CoreServicesVnet`
for connectivity testing, and two nginx web servers in `AppVnet` for Tasks 5 and 6.
Once the VMs are provisioning, you configure VNet peering and verify cross-VNet
connectivity.

### Deploy backend VMs

1. Open **Cloud Shell** (Bash) from the top-right of the portal.

   Confirm you are using the correct subscription:

   ```bash
   az account show --query "{name:name, id:id}" -o table
   ```

   If not, set it:

   ```bash
   az account set --subscription "<your-subscription-id>"
   ```

2. Run the following commands to deploy all three VMs. 

   Do the following before running the script:
   - Replace `<password>` with a complex password of your choice.
   - Replace the resource group name with the actual resource group name assigned to you in the sandbox environment (e.g., RG-Lab2-yourname).
   - Replace the VM size based on what's available in your subscription. 

   ```bash
   # NSG to allow HTTP to the AppVnet backend VMs
   az network nsg create --resource-group RG-Lab2 --name app-nsg
   az network nsg rule create \
     --resource-group RG-Lab2 --nsg-name app-nsg \
     --name AllowHTTP --priority 100 \
     --protocol Tcp --destination-port-ranges 80 --access Allow

   # CoreServicesVM in SharedServicesSubnet (used for peering verification)
   az vm create \
     --resource-group RG-Lab2 \
     --name CoreServicesVM \
     --image Ubuntu2204 \
     --vnet-name CoreServicesVnet \
     --subnet SharedServicesSubnet \
     --public-ip-address "" \
     --admin-username azureuser \
     --admin-password <password> \
     --size Standard_B2ls_v2 \
     --no-wait

   # vm0 in BackendSubnet1
   az vm create \
     --resource-group RG-Lab2 \
     --name az104-06-vm0 \
     --image Ubuntu2204 \
     --vnet-name AppVnet \
     --subnet BackendSubnet1 \
     --nsg app-nsg \
     --admin-username azureuser \
     --admin-password <password> \
     --size Standard_B2ls_v2 \
     --custom-data '#!/bin/bash
   apt-get update && apt-get install -y nginx
   echo "<h1>Hello World from az104-06-vm0</h1>" > /var/www/html/index.html
   mkdir -p /var/www/html/image /var/www/html/video
   echo "<h1>Image server - vm0</h1>" > /var/www/html/image/index.html
   echo "<h1>Video server - vm0</h1>" > /var/www/html/video/index.html
   systemctl enable nginx && systemctl start nginx' \
     --no-wait

   # vm1 in BackendSubnet2
   az vm create \
     --resource-group RG-Lab2 \
     --name az104-06-vm1 \
     --image Ubuntu2204 \
     --vnet-name AppVnet \
     --subnet BackendSubnet2 \
     --nsg app-nsg \
     --admin-username azureuser \
     --admin-password <password> \
     --size Standard_B2ls_v2 \
     --custom-data '#!/bin/bash
   apt-get update && apt-get install -y nginx
   echo "<h1>Hello World from az104-06-vm1</h1>" > /var/www/html/index.html
   mkdir -p /var/www/html/image /var/www/html/video
   echo "<h1>Image server - vm1</h1>" > /var/www/html/image/index.html
   echo "<h1>Video server - vm1</h1>" > /var/www/html/video/index.html
   systemctl enable nginx && systemctl start nginx' \
     --no-wait
   ```

3. All three VMs deploy in the background (approximately 5 minutes). Continue with the peering steps while they provision.

### Configure bidirectional peering

By default, resources in different virtual networks cannot communicate — even within
the same subscription and region. Virtual network peering creates a direct,
low-latency connection over the Microsoft backbone, not the public internet.

4. In the Azure portal, navigate to **CoreServicesVnet**.

5. Under **Settings**, select **Peerings**, then **+ Add**.

6. Fill in both sides of the peering simultaneously — Azure creates the return link automatically:

   | Parameter | Value |
   | --- | --- |
   | **Remote virtual network summary** | |
   | Peering link name | `AppVnet-to-CoreServicesVnet` |
   | Virtual network | **AppVnet (RG-Lab2)** |
   | **Remote virtual network peering settings** | |
   | Allow 'AppVnet' to access 'CoreServicesVnet' | Selected (default) |
   | Allow 'AppVnet' to receive forwarded traffic from 'CoreServicesVnet' | Selected |
   | **Local virtual network summary** | |
   | Peering link name | `CoreServicesVnet-to-AppVnet` |
   | **Local virtual network peering settings** | |
   | Allow 'CoreServicesVnet' to access 'AppVnet' | Selected (default) |
   | Allow 'CoreServicesVnet' to receive forwarded traffic from 'AppVnet' | Selected |

7. Select **Add** and wait a few seconds.

8. In the **Peerings** blade of `CoreServicesVnet`, confirm the **Peering status** shows **Connected**. Refresh if needed.

9. Navigate to **AppVnet → Peerings** and verify the reverse peering also shows **Connected**.

### Verify connectivity with Network Watcher

10. Confirm all three VMs have finished deploying:

    ```bash
    az vm list --resource-group RG-Lab2 --show-details \
      --query "[].{Name:name, State:powerState}" -o table
    ```

    Wait until all three show `VM running` before continuing.

11. Retrieve the private IP address of `CoreServicesVM`:

    ```bash
    az vm show --resource-group RG-Lab2 --name CoreServicesVM \
      --show-details --query privateIps -o tsv
    ```

    Note this IP — you will use it as the destination in the Network Watcher test.

12. Search for and select **Network Watcher**.

13. In the **Network diagnostic tools** section, select **Connection troubleshoot**.

14. Configure the test:

    | Field | Value |
    | --- | --- |
    | Source type | **Virtual machine** |
    | Virtual machine | **az104-06-vm0** |
    | Destination type | **Virtual machine** |
    | Virtual machine | **CoreServicesVM** |
    | Protocol | **TCP** |
    | Destination port | `22` |

15. Select **Run diagnostic tests**. Confirm the result shows **Reachable** — traffic from `AppVnet` is flowing to `CoreServicesVnet` over the peering.

### Verify private DNS resolution

16. Now that `CoreServicesVM` is running and auto-registration is active, confirm that an A record has been created in the private zone. In the portal, navigate to **Private DNS zones → private.adventuretravel.com → Recordsets**. You should see `coreservicesvm` with an A record pointing to its private IP.

17. Verify that `az104-06-vm0` in `AppVnet` can resolve the name (it can, because `AppVnet` is linked to the zone as a resolver). Run this from Cloud Shell:

    ```bash
    az vm run-command invoke \
      --resource-group RG-Lab2 \
      --name az104-06-vm0 \
      --command-id RunShellScript \
      --scripts "nslookup coreservicesvm.private.adventuretravel.com"
    ```

    Confirm the output resolves to `CoreServicesVM`'s private IP. This demonstrates that VMs in the peered `AppVnet` can use the private DNS zone to discover resources in `CoreServicesVnet` by name, without hardcoding IP addresses.

**Key point:** Virtual network peering is non-transitive by default. If VNet A peers with VNet B, and VNet B peers with VNet C, VNet A cannot reach VNet C through VNet B. To achieve transitive routing you need either a hub VNet with a Network Virtual Appliance, or Azure Virtual WAN.

---

## Task 5: Configure an Azure Load Balancer

Azure Load Balancer operates at Layer 4 (TCP/UDP) and distributes inbound traffic
across backend virtual machines. The Standard SKU provides zone redundancy, SLA
guarantees, and health probes — use it for all production workloads.

The `AppVnet` and backend VMs (`az104-06-vm0` and `az104-06-vm1`) were already
provisioned in Task 4. Confirm they are running before proceeding.

### Create the Load Balancer

1. In the Azure portal, search for and select **Load balancers**, then **+ Create Standard Load Balancer**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
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
   | New public IP name | `az104-lbpip` |
   | SKU | **Standard** |
   | Assignment | **Static** |
   | Routing preference | **Microsoft network** |

   Select **Save**, then **Next: Backend pools**.

4. Select **+ Add a backend pool**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-be` |
   | Virtual network | **AppVnet (RG-Lab2)** |
   | Backend pool configuration | **NIC** |

   Select **+ Add** and add both `az104-06-vm0` and `az104-06-vm1`. Select **Add**, then **Save**.

5. Leave **Inbound rules** and remaining tabs at defaults. Select **Review + create**, then **Create**. Wait for deployment.

### Add a load balancing rule

6. Navigate to the deployed `az104-lb` resource. In the **Settings** blade, select **Load balancing rules**, then **+ Add**:

   | Setting | Value |
   | --- | --- |
   | Name | `az104-lbrule` |
   | IP Version | **IPv4** |
   | Frontend IP address | **az104-fe** |
   | Backend pool | **az104-be** |
   | Protocol | **TCP** |
   | Port | `80` |
   | Backend port | `80` |
   | Health probe | **Create new** → name `az104-hp`, Protocol `TCP`, Port `80`, Interval `5` |
   | Session persistence | **None** |
   | Idle timeout (minutes) | `4` |
   | Enable TCP reset | **Disabled** |
   | Enable Floating IP | **Disabled** |

   Select **Save**.

### Test the Load Balancer

7. On the **Frontend IP configuration** blade of `az104-lb`, copy the public IP address.

8. Open a browser and navigate to `http://<frontend-ip-address>`. You should see **Hello World from az104-06-vm0** or **az104-06-vm1**.

9. Refresh the page several times (or use an InPrivate window). Confirm that responses alternate between both virtual machines — this demonstrates round-robin load distribution.

**Key point:** The Standard Load Balancer is zone-redundant and SLA-backed. The Basic SKU is for dev/test only. Health probes monitor backend instance health — if a probe fails, the Load Balancer stops sending traffic to that instance until it recovers. Session persistence (sticky sessions) can be enabled if your application requires a client to always reach the same backend.

---

## Task 6: Configure an Application Gateway with Path-Based Routing

Azure Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer. Unlike the
standard Load Balancer which routes based on IP and port, Application Gateway can
route based on the URL path — directing `/image/*` traffic to an image-serving backend
pool and `/video/*` traffic to a video-serving backend pool.

### Add the Application Gateway subnet

The Application Gateway requires a dedicated subnet of /27 or larger.

1. Navigate to **AppVnet → Subnets**, then **+ Subnet**.

   | Setting | Value |
   | --- | --- |
   | Name | `AppGwSubnet` |
   | Starting address | `10.60.3.224` |
   | Size | `/27` |

2. Select **Add**.

### Create the Application Gateway

3. Search for and select **Application gateways**, then **+ Create**.

4. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab2** |
   | Application gateway name | `az104-appgw` |
   | Region | **Australia East** |
   | Tier | **Standard V2** |
   | Enable autoscaling | **No** |
   | Instance count | `2` |
   | HTTP2 | **Disabled** |
   | Virtual network | **AppVnet** |
   | Subnet | **AppGwSubnet (10.60.3.224/27)** |

5. Select **Next: Frontends**:

   | Setting | Value |
   | --- | --- |
   | Frontend IP address type | **Public** |
   | Public IP address | **Add new** → name `az104-gwpip` |

6. Select **Next: Backends**. Add three backend pools:

   **Default pool (both VMs):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-appgwbe` |
   | Add backend pool without targets | **No** |
   | Virtual machine | **az104-06-vm0 NIC** |
   | Virtual machine | **az104-06-vm1 NIC** |

   Select **Add**, then **+ Add a backend pool** again.

   **Images pool (vm0 only):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-imagebe` |
   | Virtual machine | **az104-06-vm0 NIC** |

   Select **Add**, then **+ Add a backend pool** again.

   **Videos pool (vm1 only):**

   | Setting | Value |
   | --- | --- |
   | Name | `az104-videobe` |
   | Virtual machine | **az104-06-vm1 NIC** |

   Select **Add**.

7. Select **Next: Configuration**, then **+ Add a routing rule**:

   | Setting | Value |
   | --- | --- |
   | Rule name | `az104-gwrule` |
   | Priority | `10` |
   | **Listener tab** | |
   | Listener name | `az104-listener` |
   | Frontend IP | **Public IPv4** |
   | Protocol | **HTTP** |
   | Port | `80` |
   | Listener type | **Basic** |

8. On the **Backend targets** tab:

   | Setting | Value |
   | --- | --- |
   | Target type | **Backend pool** |
   | Backend target | `az104-appgwbe` |
   | Backend settings | **Add new** → name `az104-http`, Protocol `HTTP`, Port `80` → **Add** |

9. In the **Path-based routing** section, select **Add multiple targets to create a path-based rule** and add two path rules:

   **Image routing rule:**

   | Setting | Value |
   | --- | --- |
   | Path | `/image/*` |
   | Target name | `images` |
   | Backend settings | **az104-http** |
   | Backend target | `az104-imagebe` |

   Select **Add**, then add the second:

   **Video routing rule:**

   | Setting | Value |
   | --- | --- |
   | Path | `/video/*` |
   | Target name | `videos` |
   | Backend settings | **az104-http** |
   | Backend target | `az104-videobe` |

   Select **Add**.

10. Select **Next: Tags**, then **Next: Review + create**, then **Create**.

    > **Note:** The Application Gateway takes 5–10 minutes to deploy.

### Test path-based routing

11. Once deployed, navigate to **az104-appgw → Overview** and copy the **Frontend public IP address**.

12. Open a browser and navigate to `http://<frontend-ip>/image/`. Confirm you see the image server response from **vm0**.

13. Open a new browser window and navigate to `http://<frontend-ip>/video/`. Confirm you see the video server response from **vm1**.

14. Navigate to `http://<frontend-ip>/` (no path). Confirm the default pool responds — either vm0 or vm1.

**What just happened:** The Application Gateway inspected the URL path of each request and forwarded it to the appropriate backend pool — `/image/*` went to vm0, `/video/*` went to vm1, and unmatched paths went to the default backend pool. This is Layer 7 routing: the gateway understands HTTP, not just TCP ports.

**Key point:** Application Gateway (Standard V2) supports SSL/TLS termination, WAF (upgraded to WAF V2 tier), cookie-based session affinity, connection draining, and custom health probes. Use Application Gateway when you need URL-based or host-header-based routing. Use Load Balancer when you only need TCP/UDP distribution.

---

## Cleanup

**Remove all lab resources to avoid ongoing charges.**

1. In the Azure portal, navigate to **RG-Lab2**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name `RG-Lab2` to confirm, then select **Delete**.

   > Alternatively, from Cloud Shell:
   > ```bash
   > az group delete --name RG-Lab2 --yes --no-wait
   > ```

---

## Key Takeaways

- **Virtual networks and subnets** provide the foundation for all Azure networking.
  Avoid overlapping IP address ranges across VNets and on-premises networks from the
  outset — fixing this later requires tearing down and re-creating resources.

- **NSGs** filter traffic at the subnet or NIC level using allow/deny rules evaluated
  in priority order. Attaching an NSG to a subnet enforces rules on all traffic entering
  or leaving that subnet. In this lab, the NSG on `DatabaseSubnet` allows only `AppVnet`
  traffic on port 1433, blocking all other inbound access and outbound internet egress.

- **Azure DNS** provides both public and private DNS hosting. Public zones are
  internet-accessible; private zones are scoped to specific virtual networks. Enabling
  auto-registration on a VNet link means Azure automatically creates and maintains DNS
  records for every VM in that VNet — no manual A record management needed. Linking
  additional VNets as resolvers (without auto-registration) lets peered VMs look up
  those records across VNet boundaries.

- **Virtual network peering** connects two VNets directly over the Microsoft backbone
  with low latency. In this lab, peering `AppVnet` to `CoreServicesVnet` gives the
  application tier direct access to the database subnet without routing over the internet.
  Peering is non-transitive — hub-and-spoke topologies require either a Network Virtual
  Appliance or Azure Virtual WAN for transitive routing.

- **Azure Load Balancer (Standard)** distributes TCP/UDP traffic at Layer 4 using
  health probes and configurable rules. Use it for VM-level traffic distribution
  where the application is not HTTP/HTTPS.

- **Azure Application Gateway** routes HTTP/HTTPS traffic at Layer 7 using URL path
  and host-header rules. Use it when requests must be directed to different backends
  based on content type, API version, tenant, or any other URL attribute.

---

## Resources

- [Azure Virtual Networks documentation](https://learn.microsoft.com/en-us/azure/virtual-network/)
- [Network Security Groups overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Application Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups)
- [Azure DNS documentation](https://learn.microsoft.com/en-us/azure/dns/)
- [Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [Azure Load Balancer documentation](https://learn.microsoft.com/en-us/azure/load-balancer/)
- [Azure Application Gateway documentation](https://learn.microsoft.com/en-us/azure/application-gateway/)
- [Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/)
