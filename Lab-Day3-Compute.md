---
lab:
  title: 'Lab Day 3: Compute'
  module: 'Advanced Azure Bootcamp – Day 3'
---

# Lab Day 3 – Compute: Fundamentals & Selection

## Lab Introduction

In this lab you explore the four primary Azure compute surfaces and examine the
cost controls available on each. You will work through the following tasks:

- Deploy a virtual machine and apply cost-saving options
- Build and invoke a serverless Azure Function
- Deploy a web application using Azure App Service
- Run a containerised workload with Azure Container Apps

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab3**.
Note: You will be assigned your own unique resource group name in the sandbox environment to avoid conflicts with other participants. Create a resource group in the same region you will deploy resources to (e.g., Australia East). Name the resource group RG-Lab3-yourname. Note the resource group name for use in all lab tasks.
- Owner rights at the resource group level
- A local terminal (Bash or PowerShell) or Azure Cloud Shell

## Estimated Timing: 90 minutes

## Lab Scenario

Your organisation wants to evaluate the four main compute surfaces before
committing to a long-term architecture. As the cloud practitioner leading the
proof-of-concept, you will deploy one representative workload on each surface,
observe the operational model, and identify the relevant cost levers. At the
end of the lab you will have four running services and a clear picture of the
trade-offs between them.

## Architecture Overview

```
Subscription
└── RG-Lab3
    ├── lab3-vm  (Standard_B2s, Ubuntu 22.04)
    │   └── Auto-shutdown configured
    │
    ├── lab3-funcapp  (Flex Consumption Plan, Node.js 22)
    │   └── HttpTriggerFunction  ← HTTP GET → JSON response
    │
    ├── lab3-appserviceplan  (B1, Linux)
    │   └── lab3-webapp  ← public URL → static site (appsvc/staticsite)
    │
    └── lab3-env  (Container Apps Environment)
        └── lab3-containerapp
              image: mcr.microsoft.com/azuredocs/containerapps-helloworld
```

## Job Skills

- Task 1: Deploy a virtual machine and explore cost-saving options
- Task 2: Create and invoke a serverless Azure Function
- Task 3: Deploy a web application with Azure App Service
- Task 4: Run a containerised workload with Azure Container Apps

---

## Task 1: Deploy a Virtual Machine and Explore Cost-Saving Options

Azure Virtual Machines give you full control over the operating system, runtime,
and configuration — at the cost of the most operational overhead of any compute
surface. In this task you deploy a small Linux VM, then walk through the key
cost levers available before and after deployment.

> **Design note:** VMs are most appropriate when you need a specific OS version,
> must run licensed software, require persistent local storage, or need to lift and
> shift an existing workload without modification. For greenfield applications,
> evaluate the higher-level services in Tasks 2–4 first.

### Create the virtual machine

1. Sign in to the [Azure portal](https://portal.azure.com).

2. Search for and select **Virtual machines**, then select **+ Create → Azure virtual machine**.

3. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab3** |
   | Virtual machine name | `lab3-vm` |
   | Region | **Australia East** (keep consistent throughout the lab) |
   | Availability options | **No infrastructure redundancy required** |
   | Security type | **Standard** |
   | Image | **Ubuntu Server 24.04 LTS – x64 Gen2** |
   | VM architecture | **x64** |
   | Size | Select **See all sizes** → search `B2s` → choose **Standard_B2s** |

   > **Why B-series?** B-series (burstable) VMs accumulate CPU credits when idle
   > and spend them during bursts. They are significantly cheaper than D-series for
   > workloads with low average CPU but occasional spikes — dev/test, light web
   > servers, and CI agents are ideal candidates.

4. Under **Administrator account**, choose **Password** and enter a username and
   strong password. Note both — you will need them later.

5. Under **Inbound port rules**, set **Public inbound ports** to **None**.
   You will access the VM through a managed channel (Bastion or serial console)
   rather than exposing SSH directly.

6. Select the **Disks** tab. Note the **OS disk type** dropdown — changing from
   **Premium SSD** to **Standard SSD** or **Standard HDD** reduces storage cost
   for workloads that do not require high IOPS.

7. Select the **Management** tab. Locate **Auto-shutdown** and enable it:

   | Setting | Value |
   | --- | --- |
   | Enable auto-shutdown | **On** |
   | Shutdown time | **19:00:00** |
   | Time zone | Your local time zone |
   | Notification before shutdown | **Off** (or configure an email) |

   > **Why auto-shutdown?** Forgotten VMs are one of the largest sources of
   > wasted cloud spend. Auto-shutdown guarantees a VM is stopped (deallocated)
   > each evening so you are not charged compute while it sits idle overnight.
   > Compute charges stop when a VM is **deallocated** — not merely **stopped**
   > from within the OS, which keeps the VM allocated and still incurs cost.

8. Select **Review + create**, then **Create**. Continue with the cost exploration
   steps below while the VM deploys (~2 minutes).

### Explore cost-saving options

9.  While the deployment runs, open a new browser tab and navigate to the
    [Azure Pricing Calculator](https://azure.microsoft.com/en-au/pricing/calculator/).
    Add a **Virtual Machines** product and compare the monthly cost of
    **Standard_B2s** on Pay-As-You-Go versus a **1-Year Reserved** commitment.
    Note the approximate savings percentage — typically 30–40 %.

    > **Reserved Instances** require a 1- or 3-year upfront or monthly commitment.
    > They are ideal for any VM that will run continuously — production web servers,
    > databases, and domain controllers. Reserved capacity is applied automatically
    > to matching running VMs in your subscription; you do not change the VM itself.

10. Navigate back to **Virtual machines** and wait for `lab3-vm` to show
    **Running** status.

11. Select `lab3-vm`. On the **Overview** blade, select **Stop** to deallocate
    the VM. Confirm the shutdown. Once the status shows **Stopped (deallocated)**,
    observe on the **Availability + scale -> Size** blade that you can resize a deallocated VM to any SKU
    with no data loss.

12. Restart the VM by selecting **Start** on the **Overview** blade. Leave it
    running — you do not need to connect to it for this lab.

**Key point:** The total cost of a VM has several independent levers — SKU
selection, OS disk tier, commitment model (PAYG vs Reserved vs Spot), licensing
(Azure Hybrid Benefit), and operational discipline (auto-shutdown, right-sizing).
Addressing all of them together typically produces 50–70 % savings over an
unoptimised Pay-As-You-Go deployment.

---

## Task 2: Create and Invoke a Serverless Azure Function

Azure Functions is a serverless compute service — you deploy only your code, and
Azure handles provisioning, scaling, and availability. The **Flex Consumption plan**
charges only for executions (the first 1 million per month are free) and scales
to zero when idle, eliminating the concept of an "idle VM".

### Create the Function App

1. Search for and select **Function App**, then select **+ Create → Flex Consumption**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab3** |
   | Function App name | `lab3-funcapp-<your-initials>` (must be globally unique) |
   | Region | **Australia East** |
   | Runtime stack | **Node.js** |
   | Version | **22 LTS** |

3. Select **Review + create**, then **Create**. The deployment provisions a
   Storage Account automatically — Functions requires storage for state, logs,
   and zip-deployment artefacts.

4. Select **Go to resource** when the deployment completes.

### Create and deploy an HTTP-triggered function via Cloud Shell

> **Note:** The Flex Consumption plan does not support in-portal function creation
> or the `Code + Test` editor. Functions must be deployed via CLI, VS Code, or a
> CI/CD pipeline. In this lab you will use Azure Cloud Shell with the Azure
> Functions Core Tools.

5. Open **Azure Cloud Shell** (Bash) from the portal toolbar. Select Settings then **Go to Classic version**

6. Verify that Azure Functions Core Tools is available — it is pre-installed in Cloud Shell:

   ```bash
   func --version
   ```

   You should see a version number (4.x or higher). No installation is required.

7. Create a new Functions project in Cloud Shell:

   ```bash
   mkdir lab3-func && cd lab3-func
   func init --worker-runtime node --language javascript --model V4
   ```

8. Add an HTTP-triggered function:

   ```bash
   func new --name HttpTriggerFunction --template "HTTP trigger" --authlevel anonymous
   ```

9. Open the generated function file in the Cloud Shell editor. Since you are already inside the `lab3-func` directory from step 7, use `$(pwd)` to build the correct absolute path:

   ```bash
   code $(pwd)/src/functions/HttpTriggerFunction.js
   ```

   > **Why `$(pwd)` instead of `~/`?** Cloud Shell's `code` command prepends `$HOME` to the path internally, so passing `~/lab3-func/...` causes it to double the home directory (e.g. `/home/user/home/user/...`). Using `$(pwd)` passes the already-resolved absolute path and avoids this.

   Replace the entire contents of the file with the following, then press **Ctrl+S** to save and close the editor:

   ```javascript
   const { app } = require('@azure/functions');

   app.http('HttpTriggerFunction', {
       methods: ['GET', 'POST'],
       authLevel: 'anonymous',
       handler: async (request, context) => {
           const name = request.query.get('name') || 'World';
           return {
               body: JSON.stringify({
                   message: `Hello, ${name}! This response came from an Azure Function.`,
                   timestamp: new Date().toISOString(),
                   trigger: 'HTTP'
               }),
               headers: { 'Content-Type': 'application/json' }
           };
       }
   });
   ```

10. Deploy the function to your Function App (replace `<your-initials>` to match your app name):

    ```bash
    func azure functionapp publish lab3-funcapp-<your-initials> --node --build remote
    ```

    Wait for the deployment to complete. You will see the function URL printed at the end of the output.

### Test the function

11. Copy the function URL from the deployment output. Open a browser and navigate to:

    ```
    <function-url>?name=MYOB
    ```

    Confirm you receive a JSON response with the message `Hello, MYOB!...`.

12. Alternatively, test directly from Cloud Shell by retrieving the URL automatically (replace `<your-initials>`):

    ```bash
    FUNC_URL=$(az functionapp function show \
      --resource-group RG-Lab3 \
      --name lab3-funcapp-<your-initials> \
      --function-name HttpTriggerFunction \
      --query "invokeUrlTemplate" -o tsv)
    curl "${FUNC_URL}?name=MYOB"
    ```

    > **Note:** In Bash, variable assignments must have **no spaces** around `=`. Writing `FUNC_URL = https://...` causes the error `FUNC_URL: command not found`.

13. Back in the portal, navigate to the Function App and select **Overview** in the left nav. The `HttpTriggerFunction` now appears in the list under the Functions tab. Select it to view its details.

14. Navigate to **Settings → Scale and concurrency** on the Function App. Scroll down to the **Always-ready instance count** section — this controls how many instances are kept warm to avoid cold starts. Azure still manages all burst scaling automatically beyond that baseline.

**Key point:** The **Flex Consumption plan** is pay-per-execution with automatic scaling from zero, like the classic Consumption plan, but adds VNet integration support and configurable always-ready instances to reduce cold starts — without committing to a fixed hourly cost. The **Premium plan** provides always-warm instances and more powerful SKUs at a fixed hourly cost. **App Service plan** hosting runs Functions on dedicated compute alongside your web apps — suitable when you already have spare App Service capacity.

---

## Task 3: Deploy a Web Application with Azure App Service

Azure App Service is a fully managed platform for hosting web applications,
REST APIs, and mobile backends. Unlike Functions, App Service maintains a
persistent process — it suits long-running web servers, background workers,
and applications that manage their own state in memory.

### Create the App Service Plan and Web App

1. Search for and select **App Services**, then select **+ Create → Web App**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab3** |
   | Name | `lab3-webapp-<your-initials>` (must be globally unique) |
   | Publish | **Container** |
   | Operating system | **Linux** |
   | Region | **Australia East** |

3. Under **Pricing plans**, select **Create new** and name the plan
   `lab3-appserviceplan`. Choose the **Basic B1** SKU (1 vCore, 1.75 GB RAM).

   > **Why not Free or Shared?** Free (F1) and Shared (D1) tiers run on shared
   > infrastructure with no SLA and no custom domain HTTPS. B1 is the lowest tier
   > that provides a dedicated compute instance and is suitable for light production
   > workloads. Standard and Premium tiers add autoscale, deployment slots, and
   > VNet integration.

4. Select the **Container** tab:

   | Setting | Value |
   | --- | --- |
   | Image source | **Other container registries** |
   | Access type | **Public** |
   | Registry server URL | `https://mcr.microsoft.com` |
   | Image and tag | `appsvc/staticsite:latest` |
   | Port | `80` |

5. Select **Review + create**, then **Create**. Wait for the deployment to succeed,
   then select **Go to resource**.

### Explore the App Service

7. On the **Overview** blade, select the **Default domain** link (ending in
   `.azurewebsites.net`). A browser tab opens and, after a brief startup, shows
   the **"Welcome to nginx!"** page. Note the URL — this is a free, TLS-terminated subdomain
   provided automatically by Azure.

8. Navigate to **App Service plan → Scale out** in the left nav. Observe the three scale-out methods:

   - **Manual** — set a fixed instance count (available on all tiers, currently selected)
   - **Automatic** — platform-managed scale in/out based on HTTP traffic (requires Premium v2/v3 — greyed out on B1)
   - **Rules Based** — custom metric thresholds (requires Standard or higher — greyed out on B1)

   On B1, only Manual scaling is available. Note the instance count is set to **1**.

   > **Key difference from Functions:** App Service autoscaling must be explicitly configured — you choose the method and thresholds. Functions on the Flex Consumption plan scale automatically with no configuration required.

9. Navigate to **Deployment → Deployment slots**. Note that deployment slots
   (staging, QA, etc.) are available from the **Standard S1** tier upward.
   Slots allow you to deploy and warm up a new version before swapping it into
   production with zero downtime — a critical feature for production web apps.

   > This feature is visible but greyed out on B1. Note the SKU upgrade path for
   > a future production deployment.

10. Navigate to **Log stream**. Within a few seconds you should see
   live stdout/stderr from the running container. This is the simplest form of
   application observability — no agents required for basic log tailing.

11. Navigate to **Settings → Configuration → General settings**. Because this web app was deployed as a container, there is no runtime stack selector here — that only appears for code-based deployments. Observe the platform settings available: HTTPS only enforcement, minimum TLS version, HTTP version, and Always on.

**Key point:** App Service abstracts the OS and runtime from you, but you retain
full control over your application code, startup commands, and environment
variables. It sits between Functions (zero infrastructure concern, event-driven)
and VMs (full OS control, maximum flexibility). Use App Service for traditional
web applications and APIs that run continuously and need persistent HTTP routing.

---

## Task 4: Run a Containerised Workload with Azure Container Apps

Azure Container Apps is a serverless container platform built on top of
Kubernetes and KEDA (Kubernetes Event-Driven Autoscaling). You deploy container
images directly — Azure manages the underlying cluster, node pools, and scaling.
Unlike App Service, Container Apps supports multiple containers per app
(sidecars), DAPR, and scale-to-zero for HTTP and event-driven workloads.

### Create the Container Apps Environment

A Container Apps Environment is the shared networking and logging boundary for
one or more Container Apps — analogous to a Kubernetes namespace combined with
an AKS cluster.

1. Search for and select **Container Apps**, then select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab3** |
   | Container app name | `lab3-containerapp` |
   | Region | **Australia East** |

3. Under **Container Apps Environment**, select **Create new**:

   | Setting | Value |
   | --- | --- |
   | Environment name | `lab3-env` |
   | Zone redundancy | **Disabled** (lab purposes) |

   Select **Create**, then wait for the environment validation to pass.

4. Select the **Container** tab:

   | Setting | Value |
   | --- | --- |
   | Use quickstart image | **Unchecked** |
   | Name | `hello-world` |
   | Image source | **Docker Hub or other registries** |
   |Registry login server| `mcr.microsoft.com` |
   | Image and tag | `azuredocs/containerapps-helloworld:latest` |
   | CPU and Memory | **0.5 CPU cores, 1 Gi memory** |

5. Select the **Ingress** tab:

   | Setting | Value |
   | --- | --- |
   | Ingress | **Enabled** |
   | Ingress traffic | **Accepting traffic from anywhere** |
   | Ingress type | **HTTP** |
   | Target port | `80` |

6. Select **Review + create**, then **Create**. Wait for the deployment
   (~2 minutes).

### Invoke and scale the container

7. Once deployed, select **Go to resource**. On the **Overview** blade, select
   the **Application URL** link. Confirm the page loads over HTTPS —
   the TLS certificate is provisioned and managed automatically by Azure.

8. Navigate to **Application → Scale**. Review the scaling configuration:

   | Setting | Default value | Notes |
   | --- | --- | --- |
   | Min replicas | `0` | Scale to zero when idle — no idle cost |
   | Max replicas | `10` | Upper bound on horizontal scaling |
   | Cooldown period | `300` seconds | Wait time before scaling in |
   | Polling interval | `30` seconds | How often scale rules are evaluated |

   Under **Scale rules**, note the pre-configured **http-scaler** rule of type **HTTP scaling** — this is what triggers scale-out when HTTP traffic increases.

9. Navigate to **Application → Revisions and replicas**. Note that Container Apps
   uses a **revision model** — every deployment creates a new immutable revision.
   By default the app runs in **Single revision mode**, where all traffic goes to
   the latest revision automatically. To enable traffic splitting across multiple
   revisions (e.g., 80 % stable / 20 % canary), select **Choose Deployment mode**
   and switch to **Multiple**. You can then assign traffic percentages to each
   active revision in the **Traffic** column — a built-in blue/green and canary
   deployment capability without any additional infrastructure.

10. Select **Create new revision**. On the **Create and deploy new revision** blade:
    - Optionally enter a **Name / suffix** (e.g. `v2`) to make the revision easy to identify.
    - Leave all other settings at their defaults — the existing container image and scale settings are inherited from the current revision.
    - Select **Create**.

    Confirm the new revision appears in the list and, in single revision mode, immediately receives 100 % of traffic. This demonstrates the zero-downtime deployment workflow — the new revision is provisioned before traffic is switched.

11. In Cloud Shell (Bash), use the Azure CLI to inspect the running container:

    ```bash
    az containerapp show \
      --resource-group RG-Lab3 \
      --name lab3-containerapp \
      --query "{url:properties.configuration.ingress.fqdn, \
                replicas:properties.template.scale.maxReplicas, \
                image:properties.template.containers[0].image}" \
      -o table
    ```

    Confirm the output matches the values you configured in the portal.

12. Generate a small load burst to observe scaling. From Cloud Shell:

    ```bash
    APP_URL=$(az containerapp show \
      --resource-group RG-Lab3 \
      --name lab3-containerapp \
      --query "properties.configuration.ingress.fqdn" -o tsv)

    for i in {1..3000}; do curl -s "https://${APP_URL}" -o /dev/null & done
    wait
    echo "Done"
    ```

    Navigate back to **Revisions and replicas** and refresh after 20–30 seconds.
    You may observe the replica count increase temporarily beyond 1 as Container
    Apps responds to the burst.

**Key point:** Container Apps removes Kubernetes operational complexity while
preserving container portability. You bring a container image from any registry;
Azure handles cluster management, patching, and scaling. If you need full
Kubernetes control (custom operators, node pool configuration, GPU nodes,
admission webhooks), use AKS instead — but for the majority of containerised
web services and APIs, Container Apps provides the right trade-off.

---

## Cleanup

**Remove all lab resources to avoid ongoing charges.**

1. In the Azure portal, navigate to **RG-Lab3**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name `RG-Lab3` to confirm, then select **Delete**.

   > Alternatively, from Cloud Shell:
   > ```bash
   > az group delete --name RG-Lab3 --yes --no-wait
   > ```

---

## Compute Selection Summary

Use this table as a quick reference when choosing a compute surface:

| Criterion | Virtual Machine | Azure Functions | App Service | Container Apps |
| --- | --- | --- | --- | --- |
| **Unit of deployment** | OS image | Function code | Application code / container | Container image |
| **OS control** | Full | None | None | None |
| **Idle cost** | Yes (unless deallocated) | No (Flex Consumption) | Yes | No (scale-to-zero) |
| **Cold start** | None (always running) | Yes (~0.5–2 s) | None (warm) | Yes (~1–3 s) |
| **Scaling model** | Manual / VMSS | Automatic | Manual rules or automatic | Automatic (KEDA) |
| **Max runtime** | Unlimited | 30 min (Flex Consumption) | Unlimited | Unlimited |
| **Operational overhead** | High | Very low | Low | Low |
| **Best for** | Lift-and-shift, licensed software | Event handlers, short tasks | Traditional web apps & APIs | Microservices, containerised APIs |

---

## Key Takeaways

- **Virtual machines** offer the most flexibility and the most operational
  overhead. Cost is reduced through right-sizing (B-series for burstable
  workloads), disk tier selection, Reserved Instances for steady-state VMs,
  Spot for interruptible batch jobs, Azure Hybrid Benefit for Windows/SQL
  licences, and auto-shutdown for non-production environments.

- **Azure Functions (Flex Consumption plan)** is the lowest-cost option for
  infrequent, event-driven workloads — the first million executions per month
  are free. There is no infrastructure to manage and scaling is fully automatic.
  Flex Consumption adds VNet integration and configurable always-ready instances
  over the classic Consumption plan. The trade-off is cold starts when always-ready
  instances are set to 0, and a maximum execution duration of 30 minutes.

- **App Service** sits between serverless and VMs. The platform manages the OS
  and runtime; you manage your application. It is best suited to traditional
  web applications and REST APIs that run continuously. Key production features —
  deployment slots, autoscale, custom domains with managed certificates —  are
  available from the Standard tier upward.

- **Azure Container Apps** brings the operational simplicity of App Service to
  containerised workloads. Scale-to-zero, built-in revision traffic splitting,
  DAPR support, and KEDA-based event-driven autoscaling make it the default
  choice for new containerised services when full Kubernetes control is not
  required.

---

## What This Lab Did Not Cover

The following topics are relevant to the Day 3 curriculum but were scoped out
to keep the lab within 90 minutes:

| Topic | Why it matters | Where to go next |
| --- | --- | --- |
| **VM Scale Sets (VMSS)** | Horizontal auto-scaling of identical VMs — the VM equivalent of App Service autoscale. Required for large-scale stateless VM workloads. | Azure VMSS documentation |
| **Azure Container Instances (ACI)** | Lighter-weight one-off container runs with no environment or revision model. Simpler than Container Apps for single-container batch jobs. | ACI quickstart |
| **Autoscale rules (App Service)** | Custom CPU / memory / queue-depth thresholds that trigger scale-out on App Service, as opposed to the automatic model in Functions and Container Apps. | App Service autoscale docs |
| **Azure Kubernetes Service (AKS)** | Full Kubernetes control plane — custom operators, node pool types, GPU nodes. Covered in Day 5. | Day 5 Lab |
| **Azure Batch** | Managed large-scale parallel and HPC compute — job scheduling across hundreds of VMs. | Azure Batch overview |
| **Deployment slots (App Service)** | Zero-downtime deployments via staging → production swap. Requires Standard S1 or above. | App Service deployment slots docs |

---

## Optional Lab: Deploy a Virtual Machine Scale Set (VMSS)

This optional task extends Task 1 by replacing the single VM with a **Virtual
Machine Scale Set** — the native Azure mechanism for horizontally scaling
identical VMs. VMSS is appropriate when your workload requires more compute
than a single VM can provide, and you want Azure to automatically add or remove
instances based on demand.

> **Pre-requisite:** Complete Task 1 first. `lab3-vm` should be running in
> **RG-Lab3**.

### Create the Scale Set

1. Search for and select **Virtual machine scale sets**, then select **+ Create**.

2. On the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab3** |
   | Virtual machine scale set name | `lab3-vmss` |
   | Region | **Australia East** |
   | Availability zone | **None** (lab purposes) |
   | Orchestration mode | **Uniform** |
   | Security type | **Standard** |
   | Image | **Ubuntu Server 24.04 LTS – x64 Gen2** |
   | VM architecture | **x64** |
   | Size | **Standard_B2s_v2** (2 vCPUs, 8 GiB) |

3. Note the **Instance count** field at the top of the page — set it to `2`.
   Optionally select **Configure scaling options** to review the **Scaling configuration** panel:

   - **Predictive autoscaling** — uses ML to forecast demand and scale ahead of time (leave unchecked for this lab)
   - **Scale-In policy** — controls which instance is deleted first during a scale-in (leave as **Default**)

   Select **Save** (or **Cancel**), then close the panel.

4. Under **Administrator account**, choose **Password** and enter a username and
   strong password.

5. Select **Review + create**, then **Create**. Wait for the deployment (~3 minutes).

### Explore the Scale Set

6. Select **Go to resource**. On the **Overview** blade, note the **Instance count**
   showing 2 running instances. Each instance has its own VM resource but is
   managed as a single unit through the scale set.

7. Navigate to **Availability + scale → Scaling** in the left nav. Select **Custom autoscale**. Under the **Default** scale condition:

   - Change **Scale mode** from *Scale to a specific instance count* to **Scale based on a metric**.
   - Set **Minimum** to `1`, **Maximum** to `5`, **Default** to `2`.
   - Select **+ Add a rule** and configure a scale-out rule:

     | Setting | Value |
     | --- | --- |
     | Metric name | **Percentage CPU** |
     | Operator | **Greater than** |
     | Threshold | `75` % |
     | Action | **Increase count by** `1` |

   - Select **+ Add a rule** again and configure a scale-in rule (CPU **Less than** `25` %, **Decrease count by** `1`).
   - Select **Save**.

   Select **Run history** to see a log of any scale events (empty for a new scale set).

8. Navigate to **Instances** in the left nav to see the individual VMs in the scale set. Note the columns:
   - **Status** — running state of each instance
   - **Provisioning state** — whether the last operation succeeded
   - **Health state** — application health (requires a health probe to be configured)
   - **Latest model** — whether the instance matches the current scale set model (important after OS image updates)

   From the toolbar you can **Start**, **Stop**, **Reimage**, or **Upgrade** instances.

9. Navigate to **Operations → Upgrade policy**. The default is **Manual**,
    meaning OS image upgrades are applied instance-by-instance only when you
    trigger them. **Automatic** and **Rolling** policies allow Azure to manage
    OS upgrades with configurable batch sizes and health checks — important for
    zero-downtime patching in production.

### Simulate a scale event (optional)

10. Select the instance name (e.g. **lab3-vmss_0**) to open the individual VM resource. Navigate to **Support + troubleshooting → Serial console**. Once the console connects, log in with the username and password you set during creation and run:

    ```bash
    sudo apt-get update && sudo apt-get install -y stress-ng
    ```

    > **Tip:** If you see `Could not get lock /var/lib/dpkg/lock-frontend`, another apt process (e.g. unattended-upgrades) is running in the background. Wait 1–2 minutes and retry, or skip this step if `stress-ng` is already installed from a previous run.

    Once installed, run the stressor:

    ```bash
    stress-ng --cpu 4 --timeout 600 &
    ```

    > **Tip:** The serial console may garble pasted commands (bracketed paste mode). If you see `command not found` after pasting, type the command manually instead.

    > **Why 600 seconds?** Azure autoscale evaluates metrics over a rolling window (default ~5 minutes) before triggering a scale action. A 120-second run ends before the window closes, so CPU drops below threshold before autoscale fires. 600 seconds keeps CPU elevated long enough for the evaluation to complete.

    Return to the scale set's **Scaling** blade and select **Refresh** every
    30 seconds. Once CPU exceeds 75 % you should see the instance count increase
    from 2 to 3. After the stress tool exits, the count will decrease back toward
    the minimum after the cooldown period.

### Key point

VMSS provides the same horizontal scaling model as App Service autoscale or
Container Apps, but at the VM layer — you retain full OS control at the cost
of managing OS images, agents, and health probes yourself. For new workloads,
prefer App Service or Container Apps unless you have a specific requirement for
OS-level access.

### Cleanup

When finished, delete the scale set to avoid charges:

```bash
az vmss delete --resource-group RG-Lab3 --name lab3-vmss --yes
```

---

## Resources

- [Azure Virtual Machines documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [Azure Reserved VM Instances](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/save-compute-costs-reservations)
- [Azure Spot VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [Azure Hybrid Benefit](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/hybrid-use-benefit-licensing)
- [Azure Functions overview](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview)
- [Azure Functions hosting plans](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)
- [Azure App Service overview](https://learn.microsoft.com/en-us/azure/app-service/overview)
- [App Service deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [Azure Container Apps overview](https://learn.microsoft.com/en-us/azure/container-apps/overview)
- [KEDA – Kubernetes Event-Driven Autoscaling](https://keda.sh/)
- [Azure Container Apps vs AKS](https://learn.microsoft.com/en-us/azure/container-apps/compare-options)
