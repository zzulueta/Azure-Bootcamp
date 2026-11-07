---
lab:
  title: 'Lab Day 6: Event-Driven Architecture & Observability'
  module: 'Advanced Azure Bootcamp – Day 6'
---

# Lab Day 6 – Event-Driven Architecture & Observability

## Lab Introduction

In this lab you extend the retail order management platform from Day 5 by adding
event-driven integration and a production-grade observability stack. You will work
through the core patterns a platform engineer applies when building loosely coupled,
observable systems on Azure. You will work through the following tasks:

- Provision a Log Analytics Workspace and Application Insights instance as the
  observability foundation
- Deploy an Azure Service Bus namespace with a queue and a topic/subscription pair,
  and verify reliable message delivery
- Configure an Event Grid System Topic to capture Azure Blob Storage events and route
  them to a Service Bus queue
- Build a Logic App Standard workflow that reacts to Service Bus messages and writes
  enriched order records to Azure Table Storage
- Instrument a Python application with the Application Insights SDK and explore
  distributed traces, dependency maps, and failure analysis
- Configure diagnostic settings, metric alerts, and a shared Azure Dashboard for
  operational visibility

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab6**.
  Note: You will be assigned your own unique resource group name in the sandbox
  environment to avoid conflicts with other participants. Create a resource group in
  the same region you will deploy resources to (e.g., Australia East). Name the
  resource group RG-Lab6-yourname. Note the resource group name for use in all
  lab tasks.
- Owner rights at the resource group level
- Azure Cloud Shell (browser-based) — no local tooling installation required

## Estimated Timing: 90 minutes

## Lab Scenario

The retail order management platform is now running on AKS (Day 5). The operations
team has identified two gaps:

1. **Integration gap:** When a customer submits an order, the front-end writes a JSON
   file to Blob Storage. Downstream services — an invoice generator and a warehouse
   picker — must be notified reliably and independently. A direct HTTP call is fragile;
   if a downstream service is temporarily unavailable, the order is lost.

2. **Visibility gap:** The engineering team has no insight into request latency,
   dependency failures, or error rates across the microservices. Incidents are discovered
   by customers, not by monitoring.

Your task is to build the event-driven backbone that decouples the order pipeline, and
to layer a full observability stack on top so the platform team can detect and diagnose
issues before they reach customers.

## Architecture Overview

```
RG-Lab6 (Resource Group)
│
├── Log Analytics Workspace (logs-lab6-yourname)
│   └── Application Insights (appinsights-lab6-yourname)
│         ├── Live Metrics Stream
│         ├── Application Map (distributed trace topology)
│         └── Transaction Search (end-to-end traces)
│
├── Storage Account (orderslab6yourname)
│   └── Blob Container: orders-drop
│         └── Event Grid System Topic  ──► routes BlobCreated events
│
├── Service Bus Namespace (servicebus-lab6-yourname)      [Standard SKU]
│   ├── Queue: order-intake
│   │     └── receives BlobCreated events from Event Grid
│   └── Topic: order-notifications
│         ├── Subscription: invoice-svc
│         └── Subscription: warehouse-svc
│
├── Logic App Standard (logicapp-lab6-yourname)
│   └── Workflow: process-order
│         ├── Trigger:  Service Bus — When a message arrives in order-intake
│         ├── Action 1: Parse JSON message body
│         ├── Action 2: Forward to order-notifications Topic (fan-out)
│         ├── Action 3: Insert enriched record into Azure Table Storage
│         └── Action 4: Complete the message in order-intake queue
│
└── Storage Account – Table (orderslab6yourname)
      └── Table: OrderAudit
```

## Job Skills

- Task 1: Provision the observability foundation — Log Analytics and Application Insights
- Task 2: Deploy Azure Service Bus and verify reliable message delivery
- Task 3: Configure Event Grid to capture Blob Storage events
- Task 4: Build a Logic App workflow to process order events
- Task 5: Instrument an application with Application Insights and explore distributed tracing
- Task 6: Configure alerts, dashboards, and operational visibility

---

## Task 1: Provision the Observability Foundation

Every service you deploy in this lab will emit logs and metrics to a shared **Log
Analytics Workspace**. **Application Insights** sits on top, providing application
performance management (APM) — request rates, failure rates, dependency latency, and
distributed trace correlation.

> **Why Log Analytics first?** Log Analytics is the storage and query engine.
> Application Insights is a curated application monitoring layer that stores its
> data *inside* a Log Analytics workspace (workspace-based mode, the current default).
> Creating the workspace first lets you point both Application Insights and every
> Azure service's diagnostic settings to a single store, enabling cross-service
> queries in a single Kusto (KQL) statement.

### Create the Log Analytics Workspace

1. In the Azure portal, search for and select **Log Analytics workspaces**.

2. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Name | `logs-lab6-yourname` |
   | Region | **Australia East** |

3. Select **Review + Create**, then **Create**. Provisioning takes under a minute.

4. Once deployed, navigate to the workspace. In the left menu under **Settings**, select **Properties**. Note the **Workspace ID** shown on this pane — copy it somewhere accessible.

### Create Application Insights

5. In the Azure portal, search for and select **Application Insights**.

6. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Name | `appinsights-lab6-yourname` |
   | Region | **Australia East** |
   | Log Analytics Workspace | **logs-lab6-yourname** (the workspace you just created) |

7. Select **Review + Create**, then **Create**.

8. Once deployed, navigate to `appinsights-lab6-yourname`. On the **Overview** pane, copy the **Connection String**. Store it somewhere accessible — you will paste it into an environment variable in Task 5.

### Explore the Log Analytics query environment

9. Navigate to **logs-lab6-yourname**. In the left menu select **Logs**.

   The portal opens in **Simple mode** by default, showing a "Query history" panel.
   To enter KQL, select the **Simple mode** drop-down in the top-right corner of the
   query pane and switch to **KQL mode**. A blank text editor will appear.

10. Paste the following query into the editor and select **Run** (or press **Shift+Enter**):

    ```kusto
    union isfuzzy=true AzureActivity, AppRequests, AppDependencies
    | limit 10
    ```

    At this stage the workspace is empty — the query will return zero rows. This is
    expected. After completing subsequent tasks, you will return here to run
    cross-service queries.

    > **Kusto Query Language (KQL) basics:** KQL is a read-only query language for
    > Azure Monitor. Queries flow left-to-right through a pipe (`|`). Common operators:
    > - `where` — filter rows (equivalent to SQL WHERE)
    > - `summarize` — aggregate (equivalent to SQL GROUP BY)
    > - `project` — select columns (equivalent to SQL SELECT)
    > - `order by` — sort results
    > - `render timechart` — visualise results as a chart in the portal

---

## Task 2: Deploy Azure Service Bus and Verify Reliable Message Delivery

**Azure Service Bus** is an enterprise-grade message broker that decouples producers
from consumers. Unlike a direct HTTP call (where the caller blocks until the callee
responds), Service Bus persists the message on the broker until a consumer retrieves
and acknowledges it. If the consumer is offline or slow, the message waits — it is
never lost.

Two core primitives:

| Primitive | Behaviour | Use case |
| --- | --- | --- |
| **Queue** | First-in, first-out. Each message is delivered to exactly one consumer. | Point-to-point: one producer, one consumer. |
| **Topic + Subscriptions** | Each subscription receives a complete, independent copy of every message published to the topic. | Fan-out: one producer, multiple independent consumers. |

### Create the Service Bus Namespace

1. In the Azure portal, search for and select **Service Bus**.

2. Select **+ Create** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Namespace name | `servicebus-lab6-yourname` (must be globally unique) |
   | Region | **Australia East** |
   | Pricing tier | **Standard** |

   > **Why Standard?** Standard is required for Topics and Subscriptions. Basic tier
   > supports only Queues. Premium tier adds VNET integration, larger message sizes,
   > and geo-disaster recovery — not required for this lab.

3. Select **Review + Create**, then **Create**. Provisioning takes 1–2 minutes.

### Create the order-intake Queue

4. Navigate to the `servicebus-lab6-yourname` namespace. In the left menu, under **Entities**,
   select **Queues**.

5. Select **+ Queue** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `order-intake` |
   | Maximum delivery count | `3` |
   | Message time to live | **1 day** |
   | Lock duration | **30 seconds** |
   | Enable dead-lettering on message expiration | **Checked** |

   Select **Create**.

   > **Queue parameter explanations:**
   >
   > | Setting | Why this value |
   > | --- | --- |
   > | **Name** | The queue identifier used by producers (Event Grid) and consumers (Logic App) to send and receive messages. Must be unique within the namespace. |
   > | **Maximum delivery count** | How many times Service Bus retries delivering a message before giving up. Set to `3` — after 3 failed processing attempts, the message is moved to the dead-letter queue instead of looping forever. |
   > | **Message time to live** | How long a message waits in the queue before expiring if no consumer picks it up. `1 day` prevents stale orders from being processed long after they were submitted. |
   > | **Lock duration** | When a consumer reads a message using peek-lock, it holds an exclusive lock for this duration. Set to `30 seconds` — if the consumer crashes or times out before completing the message, the lock expires and Service Bus makes the message available for redelivery automatically. |
   > | **Enable dead-lettering on message expiration** | When checked, messages that reach their time-to-live without being consumed are moved to the dead-letter sub-queue instead of being silently deleted. This ensures no order is ever lost — expired messages can be inspected, corrected, and replayed by an operator. |

### Create the order-notifications Topic and Subscriptions

6. In the left menu, select **Topics**.

7. Select **+ Topic** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `order-notifications` |
   | Maximum topic size | **1 GB** |
   | Message time to live | **1 day** |

   Select **Create**.

8. Select the `order-notifications` topic. Under **Entities**, select **Subscriptions**.

9. Select **+ Subscription** and create two subscriptions:

   **First subscription:**

   | Setting | Value |
   | --- | --- |
   | Name | `invoice-svc` |
   | Maximum delivery count | `3` |

   Select **Create**.

   **Second subscription:**

   | Setting | Value |
   | --- | --- |
   | Name | `warehouse-svc` |
   | Maximum delivery count | `3` |

   Select **Create**.

   > **Why two subscriptions?** Topics implement the publish-subscribe pattern. When
   > a message is published to `order-notifications`, Service Bus creates an
   > independent copy for each subscription. The `invoice-svc` subscription's copy
   > is completely independent of the `warehouse-svc` subscription's copy — one
   > subscriber processing (or failing) its copy has no effect on the other.
   >
   > **Real-world analogy:** Think of `order-notifications` as a company-wide email
   > distribution list. When a new order arrives, one notification is sent — and two
   > teams each get their own private copy simultaneously:
   > - **`invoice-svc`** — the billing team's inbox: generate an invoice and take payment
   > - **`warehouse-svc`** — the warehouse team's inbox: pick, pack, and ship the items
   >
   > If the billing team is slow that day, the warehouse team doesn't wait — they're
   > working from their own copy. If the warehouse system crashes, billing still got
   > paid. Neither team interferes with the other.

### Retrieve the connection string

10. Navigate back to the `servicebus-lab6-yourname` namespace. In the left menu, under
    **Settings**, select **Shared access policies**.

11. Select **RootManageSharedAccessKey**.

12. Copy the **Primary Connection String**. You will use this in Task 4 when configuring the Service Bus connection for the Logic App.

### Send and receive a test message using Service Bus Explorer

13. In the portal, navigate to `servicebus-lab6-yourname` → **Queues** → `order-intake`. Select **Service Bus Explorer**.

14. Select the **Send message** tab. In the **Message body** field, paste the
    following JSON and select **Send**:

    ```json
    {"orderId":"ORD-001","customerId":"C-42","amount":149.99,"status":"received"}
    ```

15. Select **Peek from start**. The message body appears
    in the detail pane on the right, confirming the message is sitting in the queue
    waiting for a consumer. Peeking is non-destructive — the message remains in the
    queue.

    > **Service Bus Explorer in the portal** is the fastest way to inspect and
    > debug queue content during development. In production, messages are consumed
    > by application code or through Logic Apps connectors — not by operators peeking
    > manually.

### Enable diagnostic settings for Service Bus

16. Navigate to `servicebus-lab6-yourname`. In the left menu, under **Monitoring**, select
    **Diagnostic settings**.

17. Select **+ Add diagnostic setting** and configure:

    | Setting | Value |
    | --- | --- |
    | Diagnostic setting name | `servicebus-to-logs` |
    | Logs – Category groups | Check **allLogs** |
    | Metrics | Check **AllMetrics** |
    | Destination | **Send to Log Analytics workspace** → `logs-lab6-yourname` |

    Select **Save**.

    > After saving, all Service Bus operational logs (message sends, receives,
    > dead-letter activity) and metrics (active message count, incoming/outgoing
    > message rates) will flow into the shared Log Analytics workspace, where KQL
    > queries can correlate them with Application Insights data from Task 5.

---

## Task 3: Configure Event Grid to Capture Blob Storage Events

**Azure Event Grid** is a fully managed event routing service. It connects *event
sources* (Azure services that emit events — Blob Storage, Resource Manager, AKS, and
others) to *event handlers* (Service Bus, Azure Functions, Logic Apps, webhooks).
Unlike Service Bus (which is message-broker-centric), Event Grid is optimised for
lightweight, near-real-time event notification at scale.

### Create the Storage Account and Blob Container

1. In the Azure portal, search for and select **Storage accounts**.

2. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Storage account name | `orderslab6yourname` (lowercase, no hyphens, globally unique) |
   | Region | **Australia East** |
   | Primary service | **Azure Blob Storage or Azure Data Lake Storage** |
   | Performance | **Standard** |
   | Redundancy | **Locally-redundant storage (LRS)** |

3. Select **Review + Create**, then **Create**.

4. Once deployed, navigate to `orderslab6yourname`. In the left menu, under **Data storage**,
   select **Containers**.

5. Select **+ Add container**, name it `orders-drop` and select **Create**.

   > **orders-drop** acts as the order landing zone. The front-end application (or
   > a customer-facing API) drops a JSON file here each time an order is submitted.
   > Event Grid watches this container and fires a `BlobCreated` event the moment
   > any file lands — triggering the downstream processing pipeline automatically,
   > without the order producer needing to know who the consumers are.

### Retrieve the Storage Account Connection string

6. In the left menu, under **Security + networking**, select **Access keys**.

7. Select **Show** next to **Connection string** under key1, then copy the **Connection string** value and save it somewhere accessible (Notepad, etc.). You will use this in Task 4 when creating the Table Storage connection for the Logic App.

### Create the Event Grid System Topic

8. Navigate to `orderslab6yourname`. In the left menu, select **Events**.

9. On the **Events** pane, select **+ Event Subscription**.

10. Configure the event subscription:

   **Event Subscription Details:**

   | Setting | Value |
   | --- | --- |
   | Name | `order-blob-to-sb` |
   | Event Schema | **Event Grid Schema** |

   **Topic Details** (auto-populated — confirm these values, and provide a System Topic Name):

   | Setting | Value |
   | --- | --- |
   | Topic Type | Storage account |
   | Source Resource | orderslab6yourname |
   | System Topic Name | `events-lab6-yourname` |

   **Event Types:**

   Uncheck all and check only **Blob Created**.

   > Filtering to `BlobCreated` only means deletes, overwrites, and metadata changes
   > do not trigger the workflow — only net-new order files do.

   **Endpoint Type:**

   Select **Service Bus Queue** as the endpoint type. Select **Configure an endpoint**
   and choose:

   | Setting | Value |
   | --- | --- |
   | Subscription | your Azure subscription |
   | Resource group | **RG-Lab6** |
   | Service Bus Namespace | `servicebus-lab6-yourname` |
   | Service Bus Queue | `order-intake` |

   Select **Confirm Selection**, then **Create**.

   > **What happens now:** Every time a blob is uploaded to `orderslab6yourname`, Event
   > Grid calls the Service Bus data plane and enqueues a `BlobCreated` event message
   > into `order-intake`. The event message is a JSON envelope that includes the blob
   > URL, content length, and etag. The order pipeline can read the blob URL from the
   > event, then download the actual order JSON from Blob Storage.

### Test the end-to-end event flow

11. Create a test order file on your local machine and upload it to the `orders-drop`
   container using the portal.

   **a.** On your local machine, open any text editor (Notepad, VS Code, etc.) and
   paste the following JSON, then save the file as `order-002.json`:

   ```json
   {
     "orderId": "ORD-002",
     "customerId": "C-17",
     "items": [
       { "sku": "WIDGET-A", "qty": 3, "unitPrice": 24.99 },
       { "sku": "WIDGET-B", "qty": 1, "unitPrice": 89.99 }
     ],
     "total": 164.96,
     "submittedAt": "2026-03-21T09:15:00Z"
   }
   ```

   **b.** In the Azure portal, navigate to `orderslab6yourname` → **Containers** →
   `orders-drop`. Select **Upload**, browse to your `order-002.json` file, and
   select **Upload**.

12. Within 5–15 seconds, verify that Event Grid delivered a message to the queue.
    In the portal, navigate to `servicebus-lab6-yourname` → **Queues** → `order-intake` →
    **Service Bus Explorer** → **Peek from start**.

    You should now see **two messages** in the queue:
    - The test message sent manually in Task 2 (Step 14)
    - The `BlobCreated` event just delivered by Event Grid

    Inspect the Event Grid message body. It will be a JSON envelope similar to:

    ```json
    {
      "topic": "/subscriptions/.../resourceGroups/RG-Lab6/providers/Microsoft.Storage/storageAccounts/orderslab6yourname",
      "subject": "/blobServices/default/containers/orders-drop/blobs/order-002.json",
      "eventType": "Microsoft.Storage.BlobCreated",
      "id": "8328ff23-901e-004a-23a3-bad93b060102",
      "eventTime": "2026-03-23T09:01:10.7747937Z",
      "dataVersion": "",
      "metadataVersion": "1",
      "data": {
        "api": "PutBlob",
        "requestId": "8328ff23-901e-004a-23a3-bad93b000000",
        "eTag": "0x8DE88BABA84A18B",
        "contentType": "application/json",
        "contentLength": 250,
        "blobType": "BlockBlob",
        "accessTier": "Default",
        "url": "https://orderslab6yourname.blob.core.windows.net/orders-drop/order-002.json",
        "sequencer": "000000000000000000000000000143E20000000001a59957",
        "storageDiagnostics": {
          "batchId": "b57d099a-e006-001d-00a3-ba7708000000"
        }
      }
    }
    ```

    > **Event Grid vs Service Bus — when to use which:**
    >
    > | Dimension | Event Grid | Service Bus |
    > | --- | --- | --- |
    > | Message size | Up to 1 MB | Up to 100 MB (Premium: 100 GB) |
    > | Delivery guarantee | At-least-once | At-least-once (queue) or exactly-once with sessions |
    > | Ordering | Not guaranteed | Guaranteed within a session |
    > | Primary pattern | Reactive eventing (something happened) | Reliable command/data messaging |
    > | Fan-out | Via multiple subscriptions per topic | Via Topics |
    > | Retry policy | Configurable exponential back-off (up to 24 hours) | Lock duration + dead-letter |
    >
    > In this lab, Event Grid handles the thin notification layer (blob created →
    > enqueue an event pointer), while Service Bus handles reliable processing
    > and the fan-out to downstream services.

---

## Task 4: Build a Logic App Workflow to Process Order Events

**Azure Logic Apps** is a low-code integration platform for building automated
workflows. A Logic App **workflow** consists of a **trigger** that starts the run and
one or more **actions** that execute sequentially (or in parallel, with branching) in
response. Logic Apps Workflow Service Plan runs as a single-tenant, container-based host in an App
Service Environment, giving you network isolation and deterministic scaling.

In this task you build a workflow that:
1. Triggers when a message arrives in the `order-intake` queue
2. Parses the event body to extract the blob URL
3. Publishes a notification to the `order-notifications` topic (fan-out to `invoice-svc`
   and `warehouse-svc` subscribers)
4. Writes an audit record to Azure Table Storage
5. Completes the message to settle it from the `order-intake` queue

### Create a Storage Table for order auditing

1. Navigate to `orderslab6yourname`. In the left menu, under **Data storage**, select
   **Tables**.

2. Select **+ Table**, name it `OrderAudit`, and select **OK**.

### Create the Logic App resource

3. In the Azure portal, search for and select **Logic apps**.

4. Select **+ Create**. A **Select a hosting option** screen appears. Select **Workflow Service Plan** (under the Standard column) and select **Select**.

   > **Why Workflow Service Plan?** This is the single-tenant Standard hosting option. It provides a dedicated compute host with VNET integration, which is required for the private networking model in this lab. The Consumption (Multi-tenant) option runs on shared Microsoft infrastructure with no VNET support and is not suitable here.

5. Configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab6** |
   | Logic App name | `logicapp-lab6-yourname` |
   | Region | **Australia East** |
   | Windows Plan | (New) accept the default name |
   | Pricing plan | **WS1** (smallest, sufficient for lab) |

6. Select the **Monitoring** tab and configure:

   | Setting | Value |
   | --- | --- |
   | Enable Application Insights | **Yes** |
   | Application Insights | **appinsights-lab6-yourname** |

   > Enabling Application Insights here automatically instruments all Logic App
   > workflow runs with distributed trace context. Each run generates a dependency
   > telemetry item visible in the Application Map (Task 5).

7. Select **Review + Create**, then **Create**. Provisioning takes 2–3 minutes.

### Create the process-order workflow

8. Once deployed, navigate to `logicapp-lab6-yourname`. In the left menu, under
   **Workflows**, select **Workflows**.

9. Select **+ Create** and configure:

   | Setting | Value |
   | --- | --- |
   | Workflow name | `process-order` |
   | State type | **Stateful** |

   Select **Create**.

   > **Stateful vs Stateless:** Stateful workflows persist their run history and
   > inputs/outputs to Azure Storage, enabling full run history replay and long-running
   > (days/weeks) workflows. Stateless workflows run entirely in memory —
   > faster and cheaper, but no durable run history. Use stateless for high-throughput,
   > short-lived workflows; stateful for auditable, long-running business processes.

10. Select the `process-order` workflow and then select **Designer** to open the
    visual workflow editor.

### Add the Service Bus trigger

11. In the designer, select **Add a trigger** and search for `Service Bus`.

12. Select **When messages are available in a queue (peek-lock)**.

13. Create a new Service Bus connection:

    | Setting | Value |
    | --- | --- |
    | Connection name | `sb-connection` |
    | Authentication | **Connection String** |
    | Connection String | paste the Service Bus Primary Connection String from Task 2 Step 12 |

    Select **Create new**.

14. Configure the trigger:

    | Setting | Value |
    | --- | --- |
    | Queue name | `order-intake` |

    Under **Advanced parameters**, select **Show all** to reveal hidden settings, then set **Maximum message batch size** to `1`.

    > **Peek-lock vs Receive-and-Delete:** Peek-lock takes a temporary lock on the
    > message without removing it from the queue. The message is only deleted (settled)
    > after the Logic App explicitly completes it with a **Complete message** action at
    > the end of the workflow. If the workflow fails before completing, the lock expires
    > after 30 seconds and Service Bus makes the message available again for redelivery.
    > This guarantees at-least-once processing — no messages are lost even if Logic Apps
    > crashes mid-run.

### Add the Parse JSON action

15. Below the trigger, select **+** and then **Add an action**. Search for `Parse JSON`
    and select it.

16. Configure:

    | Field | Value |
    | --- | --- |
    | Content | Select the lightning-bolt icon and choose **Content** from the trigger's dynamic content |
    | Schema | Select **Use sample payload to generate schema** and paste: |

    > **Where does this sample come from?** This is the actual Event Grid message body delivered to the `order-intake` queue, which you can inspect in Service Bus Explorer (Task 3, Step 12) Paste your own message from there, or use the sample below.

    ```json
    {
      "topic": "/subscriptions/xxx/resourceGroups/RG-Lab6/providers/Microsoft.Storage/storageAccounts/orderslab6yourname",
      "subject": "/blobServices/default/containers/orders-drop/blobs/order-002.json",
      "eventType": "Microsoft.Storage.BlobCreated",
      "id": "7a48eca8-b01e-0005-4dc0-ba5f1f06d7cd",
      "data": {
        "api": "PutBlob",
        "requestId": "7a48eca8-b01e-0005-4dc0-ba5f1f000000",
        "eTag": "0x8DE88D8105829EC",
        "contentType": "application/json",
        "contentLength": 250,
        "blobType": "BlockBlob",
        "accessTier": "Default",
        "url": "https://orderslab6yourname.blob.core.windows.net/orders-drop/order-002.json",
        "sequencer": "000000000000000000000000000219880000000000a171a8",
        "storageDiagnostics": {
          "batchId": "78a44d18-d006-0071-00c0-ba6bef000000"
        }
      },
      "dataVersion": "",
      "metadataVersion": "1",
      "eventTime": "2026-03-23T12:31:10.1725913Z"
    }
    ```

    Select **Done** to generate the schema automatically.

### Add the Service Bus publish action (fan-out)

17. Add another action below Parse JSON. Search for `Service Bus` and select
    **Send message**.

18. Configure:

    | Setting | Value |
    | --- | --- |
    | Queue or topic name | `order-notifications` |
    | Content | select the expression editor (the `fx` icon) and enter below formula then select Add |

    ```
    json(concat('{"event":"OrderReceived","blobUrl":"', body('Parse_JSON')?['data']?['url'], '","processedAt":"', utcNow(), '"}'))
    ```

    > This expression reads the blob URL parsed from the Event Grid message envelope
    > and constructs a richer notification message for downstream subscribers. Both
    > `invoice-svc` and `warehouse-svc` subscriptions receive a copy automatically —
    > that is the publish-subscribe fan-out without any additional workflow steps.

### Add the Table Storage write action (audit)

19. Add a third action. Search for `Azure Table Storage` and select **Insert or Update
    Entity**.

20. Create a new connection:

    | Setting | Value |
    | --- | --- |
    | Connection name | `storage-connection` |
    | Authentication | **Connection string** |
    | Connection string | Paste the **Storage Account Connection string** value copied in Task 3, Step 7 |

    Select **Create new**.

21. Configure the action:

    | Setting | Value |
    | --- | --- |
    | Table Name | `OrderAudit` |
    | Entity | Replace the default JSON with the following: |

    ```json
    {
      "PartitionKey": "orders",
      "RowKey": "@{guid()}",
      "BlobUrl": "@{body('Parse_JSON')?['data']?['url']}",
      "EventTime": "@{body('Parse_JSON')?['eventTime']}",
      "ProcessedAt": "@{utcNow()}"
    }
    ```

    > The **Entity** field already contains a default `{"PartitionKey": "", "RowKey": ""}` template — replace it entirely with the JSON above. `PartitionKey` and `RowKey` are embedded directly in the entity body; there are no separate top-level fields for them in this designer.

### Add the Complete Message action

22. Add a final action. Search for `Service Bus Built-in` and select **Complete the message in a queue**.

23. Configure:

    | Setting | Value |
    | --- | --- |
    | Queue name | `order-intake` |
    | Lock token | Select the **Lock token** field, then from the dynamic content picker (lightning bolt icon) select **Lock Token** (listed under the Service Bus trigger) |

    > This action settles the message. Once completed, Service Bus permanently removes
    > it from the queue. If this action is never reached (because a previous action
    > threw an error), the lock expires and the message is redelivered — guaranteed
    > at-least-once processing.

24. Select **Save** in the designer toolbar.

### Upload a new order blob and observe the workflow run

25. Create another order file called order-003.json and upload it in orders-drop container of the orderslab6yourname storage account using the portal. The content of the file should be as below:

    ```
    {
      "orderId": "ORD-003",
      "customerId": "C-88",
      "items": [
        { "sku": "GADGET-X", "qty": 2, "unitPrice": 59.99 }
      ],
      "total": 119.98,
      "submittedAt": "2026-03-21T09:30:00Z"
    }
    ```

26. In the portal, navigate to `logicapp-lab6-yourname` → **Workflows** → `process-order`
    → **Run history** (left menu).

    Within 30–60 seconds a new run should appear with status **Succeeded**.

27. Select the run to open the run detail view. Each action is shown with its
    duration and inputs/outputs. Expand **Parse JSON** to see the parsed blob URL.
    Expand **Insert or replace entity** to confirm the Table Storage write.

28. Verify the audit record was written. In the portal, navigate to `orderslab6yourname` → **Storage Browser** → **Tables** → `OrderAudit`.

    You should see one or more rows with `PartitionKey`, `RowKey`, `BlobUrl`, `EventTime`, and `ProcessedAt` columns populated.

29. Verify the fan-out. In the portal, navigate to `servicebus-lab6-yourname` →
    **Topics** → `order-notifications` → **Subscriptions**. Both `invoice-svc`
    and `warehouse-svc` should show an **Active Message Count** greater than zero.

30. Inspect the messages delivered to one subscription using Service Bus Explorer:

    **a.** Select the `invoice-svc` subscription.

    **b.** In the left menu, select **Service Bus Explorer**.

    **c.** Select **Peek from start**. The messages appear in the list — select any one to view its body in the detail pane on the right.

    You should see the notification JSON published by the Logic App, for example:

    ```json
    {"event":"OrderReceived","blobUrl":"https://orderslab6yourname.blob.core.windows.net/orders-drop/order-003.json","processedAt":"2026-03-23T12:35:00Z"}
    ```

    > Peeking is non-destructive — the messages remain in the `invoice-svc` subscription unaffected. The `warehouse-svc` subscription holds its own independent copies.

31. Confirm that the `order-intake` queue is now empty. In the portal, navigate to `servicebus-lab6-yourname` → **Queues** → `order-intake` → **Service Bus Explorer** → **Peek from start**.

    No messages should appear — the Logic App's **Complete the message** action settled every message after successful processing, and Service Bus permanently removed them from the queue.

    > This confirms the end-to-end at-least-once guarantee in action: messages were locked (peek-lock), processed through all four actions, and only then removed. If the workflow had failed partway through, the lock would have expired and the messages would have reappeared for redelivery.

---

### Task 4 Summary

In this task you built a complete, event-driven order processing pipeline entirely without writing application code:

| Step | What you built | Why it matters |
| --- | --- | --- |
| **Storage Table** | `OrderAudit` table in `orderslab6yourname` | Persistent, queryable audit trail for every processed order |
| **Logic App (Standard)** | `logicapp-lab6-zz` on a Workflow Service Plan | Single-tenant host with dedicated compute — suitable for production workloads |
| **Stateful workflow** | `process-order` | Run history is durable; each step's inputs/outputs are persisted and replayable |
| **Service Bus trigger (peek-lock)** | Polls `order-intake` queue | Guarantees at-least-once processing — message is only removed after the workflow explicitly completes it |
| **Parse JSON action** | Extracts the Event Grid envelope | Makes structured fields (blob URL, event time) available as dynamic content for downstream steps |
| **Send message action** | Publishes to `order-notifications` topic | Fan-out: a single publish delivers an independent copy to both `invoice-svc` and `warehouse-svc` — no extra code required |
| **Insert or Update Entity action** | Writes to `OrderAudit` table | Audit record with blob URL, event time, and processing timestamp — queryable via Storage Browser |
| **Complete message action** | Settles the peek-locked message | Removes the message from `order-intake` only after all previous steps have succeeded |

**Key pattern:** The workflow acts as the *orchestrator* — it reads from one reliable channel (Service Bus queue), enriches the data, fans out to subscribers, writes an audit record, then acknowledges receipt. If any step fails, the message is automatically redelivered and the run can be inspected in Run history.

> **Could Service Bus do this without a Logic App?**
>
> Service Bus has a built-in **Auto-forwarding** feature that can relay messages from a queue (or subscription) directly to another queue or topic — no Logic App or code required. You simply enable it on the queue entity and set the forwarding destination.
>
> However, Auto-forwarding only relays the message *as-is* — no transformation, no enrichment, no side effects. In this lab the Logic App is doing significantly more:
>
> | Capability | Auto-forwarding | Logic App |
> | --- | --- | --- |
> | Fan-out to topic | ✅ | ✅ |
> | Transform / enrich message | ❌ | ✅ |
> | Write audit record (Table Storage) | ❌ | ✅ |
> | Explicit peek-lock settlement | ❌ | ✅ |
> | Run history & step-level debugging | ❌ | ✅ |
>
> **Rule of thumb:** Use Auto-forwarding for simple relay (same message, no transformation). Use a Logic App, Azure Function, or custom consumer when you need to transform, enrich, branch, or write side effects.

---

## Task 5: Instrument an Application with Application Insights and Explore Distributed Tracing

**Application Insights** provides end-to-end distributed tracing via OpenTelemetry. Each inbound request receives a unique `Operation ID`; every unit of downstream work adds a child **span** sharing that ID. The portal assembles all spans into a waterfall diagram and an **Application Map** topology that reflects actual runtime behaviour.

In this task you run a short Python script directly in Cloud Shell — no container or registry required — generate traffic to produce telemetry, then query the combined telemetry from the Python app and from the Logic App workflow deployed in Task 4.

### Run an instrumented Python application in Cloud Shell

1. Open the Cloud Shell and install the Application Insights SDK:

   ```bash
   pip install azure-monitor-opentelemetry==1.6.4 opentelemetry-instrumentation-requests==0.49b0 --quiet --user
   ```

2. Set the Application Insights connection string copied in **Task 1, Step 8**:

   ```bash
   export APPLICATIONINSIGHTS_CONNECTION_STRING="<your-connection-string>"
   ```

3. Create the application script:

   ```bash
   cat <<'EOF' > ~/order-api.py
   import os, time, random, json
   from http.server import HTTPServer, BaseHTTPRequestHandler
   from azure.monitor.opentelemetry import configure_azure_monitor
   from opentelemetry import trace
   from opentelemetry.trace import SpanKind, StatusCode

   # Set the service name before configure_azure_monitor is called.
   # Without this, the OpenTelemetry SDK defaults to "unknown_service"
   # which is what Application Map shows as the node label.
   os.environ.setdefault("OTEL_SERVICE_NAME", "order-api")

   # Initialise the Azure Monitor exporter once at startup.
   # All OpenTelemetry spans created in this process are automatically
   # batched and sent to Application Insights via this connection string.
   configure_azure_monitor(
       connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"]
   )
   tracer = trace.get_tracer(__name__)

   class OrderHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           if self.path.startswith("/order"):
               # SpanKind.SERVER marks this as an inbound request span.
               # Application Insights classifies SERVER spans as "Requests"
               # (visible in the Overview, Failures, and Search blades).
               # Without this, the span defaults to INTERNAL and appears
               # only as a "Trace", not counted in request metrics.
               with tracer.start_as_current_span("process-order", kind=SpanKind.SERVER) as span:
                   order_id = f"ORD-{random.randint(1000, 9999)}"
                   # Semantic HTTP attributes let the portal display the
                   # correct method, URL, and status code in the detail view.
                   span.set_attribute("http.method", "GET")
                   span.set_attribute("http.target", "/order")
                   span.set_attribute("order.id", order_id)
                   # Child span simulating a downstream DB call.
                   # Visible in the end-to-end waterfall for each request.
                   with tracer.start_as_current_span("query-inventory-db") as dep:
                       latency = random.uniform(0.02, 0.15)   # simulate 20-150 ms DB latency
                       time.sleep(latency)
                       dep.set_attribute("db_latency_ms", round(latency * 1000))
                   # Simulate a 15% error rate to demonstrate the Failures blade.
                   if random.random() < 0.15:
                       span.set_attribute("http.status_code", 500)
                       # StatusCode.ERROR is what Application Insights reads to set
                       # Success = false on the request. The http.status_code attribute
                       # alone is NOT enough to flip the failed-request counter.
                       span.set_status(StatusCode.ERROR, "inventory-db timeout")
                       self.send_response(500); self.end_headers()
                       self.wfile.write(json.dumps({"error": "inventory-db timeout", "orderId": order_id}).encode())
                       return
                   span.set_attribute("http.status_code", 200)
                   span.set_status(StatusCode.OK)   # explicitly mark span as successful
                   self.send_response(200); self.end_headers()
                   self.wfile.write(json.dumps({"orderId": order_id, "status": "accepted"}).encode())
               return
           self.send_response(404); self.end_headers()
       def log_message(self, format, *args): pass   # suppress default request logging to stdout

   HTTPServer(("0.0.0.0", 8090), OrderHandler).serve_forever()
   EOF
   ```

   > | Section | Purpose |
   > | --- | --- |
   > | `configure_azure_monitor(...)` | Initialises the OpenTelemetry exporter — all traces go to Application Insights |
   > | `process-order` span | Parent span wrapping the full order request; appears as a Request in the portal |
   > | `query-inventory-db` span | Child span simulating a DB call with 20–150 ms random latency |
   > | 15% HTTP 500 responses | Intentional errors to demonstrate Failures and Search in the portal |

4. Start the server in the background and send 40 requests:

   ```bash
   # Kill any leftover process on port 8090 from a previous run
   fuser -k 8090/tcp 2>/dev/null; sleep 1

   python ~/order-api.py &
   SERVER_PID=$!
   sleep 3   # allow the SDK to initialise

   for i in $(seq 1 40); do
     curl -s http://localhost:8090/order
     echo
     sleep 0.2
   done

   sleep 6   # allow the SDK to flush the final telemetry batch
   kill $SERVER_PID
   ```

   You should see a mix of `"status": "accepted"` (HTTP 200) and
   `"error": "inventory-db timeout"` (HTTP 500) responses.

### Explore Application Insights in the portal

5. In the portal, navigate to `appinsights-lab6-yourname` → **Overview**. After 1–2
   minutes the **Failed requests**, **Server response time**, and **Server requests** tiles will update.

6. Select **Investigate → Application Map**. You should see the **order-api** node
   showing the call count, error rate, and average response time.

   > **Note:** Application Map may take 5–10 minutes to populate. If it shows
   > "No data available", continue to Step 7 first and return here later.

7. Select **Investigate → Search** → **Last 30 minutes** → **View as individual items** → **Event types = Request** . Look for entries with **Result code 500** and select one (double-click) to open the end-to-end waterfall showing the `process-order` and `query-inventory-db` spans.

   > **How distributed tracing works:** The SDK generates a unique `Operation ID` for
   > each inbound request and a `Span ID` for each unit of work. Both IDs are injected
   > into downstream calls via the `traceparent` HTTP header. The portal assembles all
   > spans sharing the same `Operation ID` into a single waterfall, regardless of which
   > process emitted them.

8. Select **Investigate** → **Failures** → **Operations** tab → select **process-order**
   to see the failed request count and the top response codes (500). Select **XX Samples**
   on the right panel to drill into individual failed traces.


### Query telemetry in Log Analytics with KQL

The `logs-lab6-yourname` workspace receives telemetry from the Python app and from the
Logic App workflow connected to Application Insights in Task 4. The following queries
span both sources.

9. Navigate to `logs-lab6-yourname` → **Logs**. Find all failed requests from the
    last hour:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | where Success == false
    | project TimeGenerated, Name, ResultCode, DurationMs, OperationId
    | order by TimeGenerated desc
    ```

10. Compare average and P95 latency between successful and failed requests:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | summarize
        AvgDurationMs = avg(DurationMs),
        P95DurationMs = percentile(DurationMs, 95),
        RequestCount = count()
      by Name, Success
    | order by AvgDurationMs desc
    ```
11. Correlate request traces with their dependency spans:

    ```kusto
    AppRequests
    | where TimeGenerated > ago(1h)
    | join kind=inner (
        AppDependencies
        | where TimeGenerated > ago(1h)
        | project DepOperationId = OperationId, DepName = Name, DepDurationMs = DurationMs, DepSuccess = Success
    ) on $left.OperationId == $right.DepOperationId
    | project TimeGenerated, RequestName = Name, DepName, DurationMs, DepDurationMs, Success, DepSuccess
    | order by TimeGenerated desc
    ```

    This links each HTTP request to the `query-inventory-db` span it triggered, showing
    how much of the total latency came from the simulated database call.

12. Find Logic App workflow runs recorded by Application Insights (from Task 4):

    ```kusto
    AppRequests
    | where TimeGenerated > ago(24h)
    | where AppRoleName has "logicapp"
    | project TimeGenerated, AppRoleName, Name, Success, DurationMs
    | order by TimeGenerated desc
    ```
---

## Task 6: Configure Alerts, Dashboards, and Operational Visibility

Observability is not only about collecting data — it is about acting on it. In this
task you configure proactive alerts so the team is notified before a problem escalates,
and build a shared dashboard that gives the whole organisation a single view of
platform health.

### Create a metric alert for high error rate

1. Navigate to `appinsights-lab6-yourname`. In the left menu, under **Monitoring**, select
   **Alerts**.

2. Select **+ Create** → **Alert rule**.

3. On the **Condition** tab, select **Failed requests** in Signal name. Select it.

4. Configure the signal logic:

   | Setting | Value |
   | --- | --- |
   | Threshold type | **Static** |
   | Aggregation type | **Count** |
   | Value is | **Greater than** |
   | Unit | **Count** |
   | Threshold value | `0` |
   | Check every | **1 minute** |
   | Lookback period | **5 minutes** |

   > A threshold of 0 means the alert fires on any single failure — ideal for the
   > lab because it triggers immediately when you generate errors with the Python
   > script. In production, calibrate the threshold to the expected baseline error
   > rate.

5. Select **Next: Actions**. Under **Select actions**, choose **Use quick actions
   (preview)**. A panel appears on the right — fill in:

   | Setting | Value |
   | --- | --- |
   | Action group name | `lab6-ops-team` |
   | Display name | `Lab6-Ops` |

   Under **Actions**, tick **Email** and enter your own email address.

   Select **Save**.

6. Go to the **Details** tab. Set:

   | Setting | Value |
   | --- | --- |
   | Severity | **1 – Error** |
   | Alert rule name | `High error rate — order-api` |

   Select **Review + Create** → **Create**.

### Availability tests (READ-ONLY)

7. Navigate to `appinsights-lab6-yourname` → **Availability** (left menu).

   The **Availability** blade is where you would configure synthetic health probes
   that Azure runs from multiple global locations on a schedule. Each probe sends
   an HTTP request to a public URL and records whether it returned the expected
   status code within a timeout.

   > **Why we skip this step:** The Python API (`order-api.py`) runs inside your
   > Cloud Shell session and is only reachable at `http://localhost:8090`. Azure's
   > global test agents are external to Cloud Shell — they cannot reach a
   > `localhost` address. Availability tests require a **publicly accessible** HTTPS
   > endpoint (e.g., an Azure App Service, Container App, or VM with a public IP).
   >
   > In a real deployment you would select **+ Add Standard test** and configure:
   >
   > | Setting | Example value |
   > | --- | --- |
   > | Test name | `order-api-health` |
   > | URL | `https://order-api.yourdomain.com/health` |
   > | Enable retries for availability test failures | ✅ (checked) |
   > | Enable SSL certificate validity | ✅ (checked) |
   > | Test frequency | **5 minutes** |
   > | Test locations | 5 selected (e.g. Australia East, Southeast Asia, East US) |
   > | HTTP Request verb | **GET** |
   > | Success criteria – HTTP response | `200` |
   > | Test timeout | **120 seconds** |
   > | Alerts | **Enabled** |
   >
   > If fewer than the configured number of locations succeed, an availability
   > alert fires — catching regional outages that single-region monitoring misses.

8. Observe the **Availability** blade. Because no test is configured it will be
   empty. In a real deployment, once the test is set up and running, this blade provides a
   global view of the service's availability, showing which test locations are succeeding.

### Create a Log Analytics alert for Service Bus dead-letter messages

9. Navigate to **logs-lab6-yourname** → **Monitoring** → **Alerts** → **+ Create** → **Alert rule**.

10. On the **Condition** tab, click the **Signal name** dropdown. If **Custom log search** does not appear in the list, select **See all signals**, search for `Custom log search`, and select it. 

    In the upper right, select **KQL mode**. In the query editor that appears, enter the KQL query below and select **Run query** to validate it returns results:

    ```kusto
    AzureMetrics
    | where ResourceProvider == "MICROSOFT.SERVICEBUS"
    | where MetricName == "DeadletteredMessages"
    | where Total > 0
    | summarize DeadLetterCount = sum(Total) by bin(TimeGenerated, 5m), Resource
    ```

Select **Continue Editing Alert** found at the bottom.

11. Scroll down and set the **Alert logic**:

    | Setting | Value |
    | --- | --- |
    | Operator | **Greater than** |
    | Threshold value | `0` |
    | Frequency of evaluation | **5 minutes** |

12. On the **Actions** tab, Select **Use action groups** and select the `lab6-ops-team` action group. In the **Email subject** enter `Dead-letter messages detected — Service Bus`.

    > **Why dead-letter alerts matter:** A dead-letter message is a message that
    > failed processing `Maximum delivery count` times. It means the workflow (Logic
    > App) crashed repeatably on a particular message. Without a DLQ alert, these
    > failures are silent — the order disappears from the queue but was never
    > processed. An alert ensures the team investigates before the backlog grows.

13. On the **Details** tab, set:

    | Setting | Value |
    | --- | --- |
    | Severity | **1 – Error** |
    | Alert rule name | `Service Bus dead-letter messages alert` |

    Select **Review + Create** → **Create**.

### Build an Azure Dashboard

14. In the Azure portal, select the **Dashboard** icon from the Show portal menu icon (the hamburger icon). Select **+ Create** → **Custom**.

15. Name the dashboard `Lab6 — Platform Health`.

16. In the tile gallery on the right, select **Metrics chart** and select **Add**. Repeat until you have **3 Metrics chart tiles** on the canvas. Select **Save** to persist the dashboard layout before configuring each tile.

17. Select **Edit in Metrics** on the **first tile** (Event Grid — delivery health):

    - **Scope:** Select the Event Grid System Topic created in `events-lab6-yourname`. Then select **Apply**.
    - For **Metric** select `Published Events` with **Aggregation:** `Sum`
    - Select **+ Add metric**, then for **Metric** select `Delivery Failed Events` — **Aggregation:** `Sum`
    - Select the **pencil icon** next to the auto-generated chart title at the top and rename it to `Event Grid — Delivery Health`

    Select **Save to dashboard**.

18. Select **Edit in Metrics** on the **second tile** (Service Bus Messages):

    - **Scope:** `servicebus-lab6-yourname`
    - **Metric Namespace:** `Service Bus standard metrics`
    - **Metric:** `Count of active messages in a Queue/Topic` — **Aggregation:** `Avg`
    - Select **+ Add metric**: `Count of dead-lettered messages in a Queue/Topic` — **Aggregation:** `Avg`
    - Select the **pencil icon** next to the chart title and rename it to `Service Bus Messages`

    Select **Save to dashboard**.

19. Select **Edit in Metrics** on the **third tile** (Logic App — workflow health):

    - **Scope:** `logicapp-lab6-yourname`
    - **Metric:** `Workflow Runs Completed Count` — **Aggregation:** `Sum`
    - Select **+ Add metric**, same namespace: `Workflow Runs Failure Rate` — **Aggregation:** `Sum`
    - Select the **pencil icon** next to the chart title and rename it to `Logic App — Workflow Health`

    Select **Save to dashboard**.

    > The three tiles follow the pipeline from left to right — **Event Grid** publishes an event → **Service Bus** queues the message → **Logic App** processes it. A failure or zero value in any tile pinpoints exactly which stage stalled.

20. **Save** the dashboard.

### Validate the full end-to-end telemetry chain

21. Trigger the complete pipeline one more time and confirm all observability layers captured it. Create a new order file called order-004.json and upload it in orders-drop container of the orderslab6yourname storage account using the portal. The content of the file should be as below:

    ```
    {
      "orderId": "ORD-004",
      "customerId": "C-99",
      "items": [
        { "sku": "GADGET-Z", "qty": 1, "unitPrice": 299.99 }
      ],
      "total": 299.99,
      "submittedAt": "2026-03-23T12:00:00Z"
    }
    ```

22. Confirm all observability layers captured the activity:

    | Layer | Where to check | Expected result |
    | --- | --- | --- |
    | Event Grid delivery | `events-lab6-yourname` → **Monitoring** → **Metrics** → **Published Events** | Shows 1 event published |
    | Service Bus message | `servicebus-lab6-yourname` → **Overview** → **Monitoring** → **Messages** | Messages count increases |
    | Logic App run | `logicapp-lab6-yourname` → Workflows → `process-order` → **Run history** | New Succeeded run for order-004 |
    | Dashboard | **Lab6 — Platform Health** | **Event Grid — Delivery Health** shows 1 Published Event; **Service Bus Messages** shows 2 additional Active Messages; **Logic App — Workflow Health** shows 1 Completed Run |

    Note: The steps above goes through individual service blades to validate each layer. In a real incident, you would likely start from the dashboard, identify which stage failed, then drill into that service's blade for deeper investigation.

---

## Cleanup

**Note:** Azure Service Bus (Standard), Logic Apps (Standard) and Application Insights all incur ongoing charges. Delete resources promptly after the lab.

1. In the portal, navigate to **RG-Lab6**.

2. Select **Delete resource group**.

3. Copy and paste the resource group name to confirm deletion.

4. Select **Delete**.

---

## Key Takeaways

- **Event Grid decouples event producers from consumers.** When a blob lands in
  storage, Event Grid fires an event to Service Bus without the uploading application
  needing to know which downstream services consume it. Adding a new consumer is
  a matter of adding a new Event Grid subscription — no application code changes.

- **Service Bus provides reliable, ordered, durable messaging.** The peek-lock
  pattern ensures messages are never lost, even if Logic Apps crashes mid-processing.
  The dead-letter queue captures poison messages for investigation without stalling
  the rest of the pipeline.

- **Topics + Subscriptions implement fan-out without coupling.** Publishing a single
  message to the `order-notifications` topic delivers an independent copy to both
  `invoice-svc` and `warehouse-svc`. Each subscriber processes at its own pace and
  failure rate without affecting the other.

- **Logic Apps orchestrates integration without custom code.** The visual designer
  and 400+ built-in connectors (Service Bus, Blob Storage, Table Storage, HTTP,
  Office 365, Salesforce, and more) allow platform engineers to wire systems together
  declaratively. Stateful workflow runs are durably persisted and fully auditable.

- **A workspace-based Application Insights instance centralises all telemetry in
  Log Analytics.** This allows KQL queries to join application traces (`AppRequests`,
  `AppDependencies`) with infrastructure logs (`AzureActivity`, `AzureMetrics`) in
  a single query — correlation across the full stack, not just within one service.

- **Distributed tracing (OpenTelemetry + W3C traceparent) makes microservice
  failures diagnosable.** The Application Map shows the live runtime topology.
  The end-to-end transaction view pinpoints which service and which dependency
  caused a failure. Without tracing, a 500 error at the API gateway tells you
  something is wrong but not where or why.

- **Alerts must be proactive, not reactive.** Metric alerts on failed requests and
  KQL-based alerts on dead-letter messages notify the team before customers report
  failures. Availability tests from multiple global locations catch regional outages
  that single-region monitoring misses.

- **Observability is a platform capability, not an application feature.** By
  routing diagnostic settings from every Azure service into a single Log Analytics
  workspace and pointing Application Insights at the same workspace, the platform
  team gains a unified, queryable view of the entire system — independent of which
  team owns each service. This is the foundation of a mature Site Reliability
  Engineering (SRE) practice.
