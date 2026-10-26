---
lab:
  title: 'Lab Day 7: Streaming with Apache Kafka on Azure Event Hubs'
  module: 'Advanced Azure Bootcamp – Day 7'
---

# Lab Day 7 – Streaming with Apache Kafka on Azure Event Hubs

## Lab Introduction

In Day 6 you built a reliable, event-driven order pipeline with Event Grid, Service Bus
and Logic Apps. In this lab you extend the retail platform with a **high-volume
streaming backbone** by using **Azure Event Hubs with its Apache Kafka endpoint**. Existing
Kafka applications connect to Event Hubs by changing configuration only, so the team
no longer has to run and patch a Kafka cluster. You will work through the following tasks:

- Provision a Kafka-enabled Azure Event Hubs namespace and an event hub (a Kafka topic)
- Connect standard Kafka producers and consumers using a Shared Access Signature (SAS)
  and observe that only configuration changes
- Replace shared keys with Microsoft Entra ID (OAuth 2.0) and Azure RBAC, then disable
  SAS authentication
- Explore partitions, ordering, consumer groups, offset replay and idempotent processing
- Read Kafka-produced events over AMQP with the native Azure SDK and archive them with
  Event Hubs Capture
- Deploy a serverless Azure Function that consumes the stream with a managed identity
  and applies a simple fraud-detection rule
- Decide which workloads belong on Event Hubs, Service Bus or Event Grid

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab7**.
  Note: You will be assigned your own unique resource group name in the sandbox
  environment to avoid conflicts with other participants. Create a resource group in
  the same region you will deploy resources to (e.g., Australia East). Name the
  resource group RG-Lab7-yourname. Note the resource group name for use in all
  lab tasks.
- Owner rights at the resource group level (required to create role assignments in Task 3)
- Azure Cloud Shell (**Bash**, browser-based) — no local tooling installation required
- Basic familiarity with Day 6 (Service Bus and Event Grid), which is referenced in Task 7

## Estimated Timing: 75 minutes

Task 1: Provision Kafka-enabled Event Hubs
Task 2: Produce and consume with SAS
Task 3: Switch to Microsoft Entra ID (OAuth 2.0)
Task 4: Partitions, consumer groups and idempotency
Task 5: Multi-protocol access and Capture
Task 6: Consume the stream with an Azure Function
Task 7: Choose the right service, then clean up


## Lab Scenario

The retail platform from Days 5 and 6 is preparing for its biggest flash sale of the
year. Order events are handled well by Service Bus, but the business now also wants to
capture **clickstream, cart activity and warehouse-scanner telemetry** at tens of
thousands of events per second, and feed it to analytics and fraud-detection systems.

Three constraints shape the solution:

1. **Existing Kafka investment:** The data engineering team already has Kafka producers
   and consumers that run against a self-managed cluster on-premises. Patching brokers,
   managing disks and rebalancing during peak load is a burden, but the team will not
   rewrite working applications.

2. **Security finding:** The security team found that the Kafka clients authenticate with
   shared keys embedded in configuration files. They want identity-based access with
   no shared secrets.

3. **Multiple consumers:** The analytics team and the fraud team must each process every
   event independently. The analytics team also prefers native Azure tooling over
   Kafka tooling, and the compliance team wants a cheap long-term archive.

Your task is to stand up a managed, Kafka-compatible ingestion endpoint on Azure Event
Hubs, secure it with Microsoft Entra ID, prove that parallel and independent consumption
works, show that non-Kafka consumers can use the same data, and then decide which parts
of the platform belong on which messaging service.

> **Is this a real Kafka cluster?** No. Event Hubs is a fully managed Azure service that exposes
> an Apache Kafka endpoint, so there is no Kafka cluster or broker for you to set up or configure.
> It supports the Kafka protocol (Kafka 1.0 and later) on port 9093. The producer and consumer
> code in this lab uses the standard `confluent-kafka` client library, which connects to
> it as it would to Kafka.

## Architecture Overview

```
RG-Lab7 (Resource Group)
│
├── Event Hubs Namespace (ehns-lab7-yourname)         [Standard, 1 TU]
│   │     Kafka endpoint: ehns-lab7-yourname.servicebus.windows.net:9093
│   │
│   └── Event Hub = Kafka topic: orders-stream        [4 partitions]
│         ├── Consumer Group: analytics
│         ├── Consumer Group: fraud-detection
│         ├── Consumer Group: audit-replay
│         ├── Consumer Group: fraud-function
│         └── Capture ──► Blob container: orders-capture (Avro files)
│
├── Storage Account (stlab7yourname)                  [Capture archive]
│
├── Function App (func-lab7-yourname)                 [Flex Consumption, Python]
│   ├── Managed identity ──► Azure Event Hubs Data Receiver (no SAS)
│   └── Function: FraudDetector  (Event Hubs trigger, consumer group fraud-function)
│
├── Storage Account (stfunclab7yourname)              [Function host and checkpoints]
│
└── Cloud Shell clients
      ├── producer.py   (Kafka protocol, SAS or OAuth)
      ├── consumer.py   (Kafka protocol, consumer groups, idempotent processing)
      └── amqp_reader.py (native azure-eventhub SDK, AMQP protocol)
```

## Job Skills

- Task 1: Provision Kafka-enabled Event Hubs and map Kafka concepts to Event Hubs concepts
- Task 2: Connect Kafka clients to Event Hubs using SAS authentication
- Task 3: Secure the Kafka endpoint with Microsoft Entra ID and Azure RBAC
- Task 4: Work with partitions, consumer groups, offsets and idempotent processing
- Task 5: Consume Kafka events over AMQP and archive them with Capture
- Task 6: Build a serverless Event Hubs consumer with Azure Functions and a managed identity
- Task 7: Select the appropriate Azure messaging service for each workload

---

## Task 1: Provision Kafka-enabled Event Hubs

Conceptually, Apache Kafka and Event Hubs are very similar. Both are **partitioned logs**
built for streaming data, in which the client controls which part of the retained log
it reads.

| Apache Kafka | Azure Event Hubs |
| --- | --- |
| Cluster | Namespace |
| Topic | Event hub |
| Partition | Partition |
| Consumer group | Consumer group |
| Offset | Offset |

> **There is no Kafka toggle.** The Kafka endpoint is available automatically on every
> **Standard, Premium and Dedicated** namespace. The **Basic** tier does not support
> the Kafka protocol.

### Create the Event Hubs namespace

1. In the Azure portal, search for and select **Event Hubs**.

2. Select **+ Create** and configure the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab7** |
   | Namespace name | `ehns-lab7-yourname` (must be globally unique) |
   | Location | **Australia East** |
   | Pricing tier | **Standard** |
   | Throughput Units | `1` |
   | Enable Auto-Inflate | **Unchecked** |

   > **Why Standard?** Standard is the lowest tier with Kafka protocol support. Kafka
   > compression (gzip), Kafka Streams and Kafka transactions need **Premium** or
   > **Dedicated**, and Streams and transactions are still in preview, so they are not used in this lab.
   >
   > **Scale model:** There are no brokers to size. Capacity is controlled by
   > throughput units (Standard) or processing units (Premium). Auto-Inflate can raise
   > throughput units automatically, and it also works with Kafka traffic.

3. Select **Review + Create**, then **Create**. Provisioning takes 1–2 minutes.

### Create the orders-stream event hub

4. Open the `ehns-lab7-yourname` namespace. In the left menu, under **Entities**, select **Event Hubs**.

5. Select **+ Event Hub** and configure:

   | Setting | Value |
   | --- | --- |
   | Name | `orders-stream` |
   | Partition count | `4` |
   | Cleanup policy | **Delete** |
   | Retention time (hrs) | `24` |

   Select **Review + create**, then **Create**.

   > **Event hub = Kafka topic.** Kafka clients see this event hub as a topic called
   > `orders-stream`. You create it in Azure, not with Kafka admin tools.
   >
   > **Partitions** are the unit of parallelism. Events with the same key always go to
   > the same partition, which preserves order for that key. The number of consumers
   > that can read a topic in parallel within one group cannot exceed the partition count.

### Create the consumer groups

6. Open the `orders-stream` event hub. In the left menu, under **Entities**, select **Consumer groups**.

7. Select **+ Consumer group** and create each of the following:

   - `analytics`
   - `fraud-detection`
   - `audit-replay`

   > Each consumer group tracks its own offsets, so each group reads the **whole** event stream
   > independently. This is how the analytics and fraud teams can process every event
   > without affecting each other.

8. On the namespace **Overview** pane, note the **Host name**
   (`ehns-lab7-yourname.servicebus.windows.net`). This, with port `9093`, is your Kafka
   `bootstrap.servers` address. Unlike a Kafka cluster, which needs access to every broker,
   Event Hubs exposes a **single stable endpoint** for all partitions.

---

## Task 2: Produce and Consume with SAS

The quickest way to connect a Kafka client is a **Shared Access Signature (SAS)**
connection string. Kafka's `PLAIN` SASL mechanism over TLS carries the connection string as the password.
In this task you connect with SAS to see how little changes, and in Task 3 you replace it.

### Prepare Cloud Shell

1. Open Cloud Shell (**Bash**) and set variables. Replace `yourname` with your own value.

   ```bash
   RG="RG-Lab7-yourname"
   NS="ehns-lab7-yourname"
   HUB="orders-stream"
   export EH_NAMESPACE="$NS.servicebus.windows.net"
   ```

2. Install the client libraries:

   ```bash
   pip install confluent-kafka azure-identity azure-eventhub --quiet --user
   ```

3. Create a least-privilege SAS policy for the application and read its connection string:

   ```bash
   az eventhubs namespace authorization-rule create \
     --resource-group $RG --namespace-name $NS \
     --name kafka-app --rights Send Listen

   export EH_CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
     --resource-group $RG --namespace-name $NS \
     --name kafka-app --query primaryConnectionString -o tsv)
   ```

   > **Why not `RootManageSharedAccessKey`?** It has Manage rights. A client that only
   > sends and listens should only hold Send and Listen rights.
   >
   > **Connection string only.** The Kafka endpoint accepts the namespace connection string as the
   > SASL password. Generated SAS *tokens* are not supported on the Kafka endpoint.

### Create the Kafka client scripts

4. Create a working folder and a shared configuration module:

   ```bash
   mkdir -p ~/lab7 && cd ~/lab7

   cat <<'EOF' > kafka_common.py
   import os
   from azure.identity import DefaultAzureCredential

   TOPIC = "orders-stream"


   def kafka_conf():
       ns = os.environ["EH_NAMESPACE"]
       conf = {
           "bootstrap.servers": f"{ns}:9093",     # the Event Hubs Kafka endpoint
           "security.protocol": "SASL_SSL",        # TLS is mandatory
           "socket.keepalive.enable": True,
           "error_cb": lambda e: print("KAFKA ERROR:", e, flush=True),
       }
       if os.environ.get("AUTH", "sas") == "oauth":
           cred = DefaultAzureCredential()

           def oauth_cb(_config):
               # confluent-kafka expects (token, expiry in seconds since epoch)
               token = cred.get_token(f"https://{ns}/.default")
               return token.token, token.expires_on

           conf.update({"sasl.mechanism": "OAUTHBEARER", "oauth_cb": oauth_cb})
       else:
           conf.update({
               "sasl.mechanism": "PLAIN",
               "sasl.username": "$ConnectionString",
               "sasl.password": os.environ["EH_CONNECTION_STRING"],
           })
       return conf
   EOF
   ```

   > | Setting | Purpose |
   > | --- | --- |
   > | `bootstrap.servers` | The only value that points at Event Hubs instead of your own cluster |
   > | `security.protocol=SASL_SSL` | Required. All Event Hubs traffic is TLS-encrypted |
   > | `sasl.mechanism=PLAIN` + `$ConnectionString` | SAS authentication. The literal user name is `$ConnectionString`; the password is the connection string |
   > | `sasl.mechanism=OAUTHBEARER` | Microsoft Entra ID authentication, used in Task 3 |
   >
   > The `AUTH` environment variable switches the authentication method without
   > changing any producer or consumer logic.

5. Create the producer:

   ```bash
   cat <<'EOF' > producer.py
   import json, random, sys, time, uuid
   from confluent_kafka import Producer
   from kafka_common import kafka_conf, TOPIC

   count = int(sys.argv[1]) if len(sys.argv) > 1 else 20
   resend_last_three = len(sys.argv) > 2 and sys.argv[2] == "dup"

   conf = kafka_conf()
   conf["enable.idempotence"] = True    # idempotent producer is supported
   producer = Producer(conf)


   def on_delivery(err, msg):
       if err:
           print("FAILED:", err, flush=True)
       else:
           print(f"key={msg.key().decode()} partition={msg.partition()} offset={msg.offset()}", flush=True)


   customers = ["C-100", "C-200", "C-300", "C-400", "C-500"]
   skus = ["SKU-TSHIRT", "SKU-MUG", "SKU-HOODIE", "SKU-CAP"]
   sent = []
   for _ in range(count):
       cust = random.choice(customers)
       event = {
           "eventId": str(uuid.uuid4()),
           "customerId": cust,
           "sku": random.choice(skus),
           "qty": random.randint(1, 3),
           "ts": time.time(),
       }
       sent.append((cust, event))
       producer.produce(TOPIC, key=cust, value=json.dumps(event), callback=on_delivery)
       producer.poll(0)

   if resend_last_three:
       # Simulates an application-level retry that re-sends events with the same eventId
       for cust, event in sent[-3:]:
           producer.produce(TOPIC, key=cust, value=json.dumps(event), callback=on_delivery)
           producer.poll(0)

   producer.flush(30)
   EOF
   ```

6. Create the consumer:

   ```bash
   cat <<'EOF' > consumer.py
   import json, sys, time
   from confluent_kafka import Consumer
   from kafka_common import kafka_conf, TOPIC

   group = sys.argv[1]
   idle_limit = int(sys.argv[2]) if len(sys.argv) > 2 else 30

   conf = kafka_conf()
   conf.update({
       "group.id": group,
       "auto.offset.reset": "earliest",   # new groups start from the oldest retained event
       "enable.auto.commit": False,       # commit only after the event is processed
       "session.timeout.ms": 30000,
   })
   consumer = Consumer(conf)
   state = {"last_activity": time.time()}


   def on_assign(_consumer, partitions):
       state["last_activity"] = time.time()
       print("ASSIGNED partitions:", sorted(p.partition for p in partitions), flush=True)


   def on_revoke(_consumer, partitions):
       print("REVOKED partitions:", sorted(p.partition for p in partitions), flush=True)


   consumer.subscribe([TOPIC], on_assign=on_assign, on_revoke=on_revoke)

   seen = set()   # in-memory only; production systems use a durable store
   processed = duplicates = 0
   try:
       while time.time() - state["last_activity"] < idle_limit:
           msg = consumer.poll(1.0)
           if msg is None:
               continue
           if msg.error():
               print("ERROR:", msg.error(), flush=True)
               continue
           state["last_activity"] = time.time()
           event = json.loads(msg.value())
           if event["eventId"] in seen:
               duplicates += 1
               print(f"DUPLICATE skipped eventId={event['eventId'][:8]}", flush=True)
           else:
               seen.add(event["eventId"])
               processed += 1
               print(f"MSG partition={msg.partition()} offset={msg.offset()} "
                     f"key={msg.key().decode()} sku={event['sku']}", flush=True)
           consumer.commit(msg=msg, asynchronous=False)
   finally:
       consumer.close()
       print(f"DONE group={group} processed={processed} duplicates={duplicates}", flush=True)
   EOF
   ```

   > **At-least-once delivery.** Event Hubs guarantees events are delivered **at least once**,
   > so a consumer can receive the same event more than once. Committing offsets **after**
   > processing, and de-duplicating on `eventId`, is the idempotent-consumer pattern you
   > will test in Task 4.

### Send and receive events

7. Send 20 events:

   ```bash
   python producer.py 20
   ```

   You should see 20 lines such as `key=C-300 partition=2 offset=0`, spread across partitions 0–3.

8. Read them with the `analytics` consumer group:

   ```bash
   python consumer.py analytics 30
   ```

   The consumer prints its `ASSIGNED partitions: [0, 1, 2, 3]`, then 20 `MSG` lines, and ends with
   `DONE group=analytics processed=20 duplicates=0` after 30 seconds without new events.

9. In the portal, open the `orders-stream` event hub → **Overview**. After 1–2 minutes, the
   **Messages** charts show 20 incoming messages and 20 outgoing messages.

   > **What changed from a normal Kafka client?** Only `bootstrap.servers` and the three
   > SASL lines. The producer and consumer logic is plain Kafka code.

---

## Task 3: Switch to Microsoft Entra ID (OAuth 2.0)

SAS keys are shared secrets: anyone holding the string can use it, it must be stored
somewhere, and rotating it affects every client. (Also, when a SAS key is regenerated
on the Kafka endpoint, **existing connections are not disconnected**.) With **Microsoft
Entra ID**, clients prove *who they are* and **Azure RBAC** decides what they may do.

### Grant data-plane roles

1. Assign yourself the data-plane roles at namespace scope:

   ```bash
   NS_ID=$(az eventhubs namespace show -g $RG -n $NS --query id -o tsv)
   ME=$(az ad signed-in-user show --query id -o tsv)

   az role assignment create --assignee-object-id $ME --assignee-principal-type User \
     --role "Azure Event Hubs Data Sender" --scope $NS_ID
   az role assignment create --assignee-object-id $ME --assignee-principal-type User \
     --role "Azure Event Hubs Data Receiver" --scope $NS_ID
   ```

   > **Owner is not enough.** Owner is a *management-plane* role. Sending and receiving
   > are *data-plane* actions that need the Data Sender and Data Receiver roles. If
   > `az ad signed-in-user show` is blocked in your sandbox, assign both roles in the portal:
   > namespace → **Access control (IAM)** → **Add role assignment**.

2. Wait about 1–2 minutes for the role assignments to propagate.

### Run the Kafka clients with OAuth

3. Switch both clients to `OAUTHBEARER`:

   ```bash
   cd ~/lab7
   export AUTH=oauth
   python producer.py 10
   python consumer.py analytics 30
   ```

   The producer prints 10 delivery lines and the consumer processes the 10 new events.
   If you see an authorization error, wait another minute and retry.

   > **How it works.** `consumer.py` and `producer.py` call `DefaultAzureCredential`, which in
   > Cloud Shell uses your signed-in identity, and request a token for
   > `https://<namespace>.servicebus.windows.net/.default`. On an Azure VM, App Service or AKS
   > pod, the same code would use a managed identity, so **no secret exists anywhere**.

### Disable SAS authentication

4. Turn off local (key-based) authentication on the namespace:

   ```bash
   az eventhubs namespace update -g $RG -n $NS --disable-local-auth true
   ```

5. Wait about 1 minute, then try the SAS-based client:

   ```bash
   AUTH=sas timeout 40 python producer.py 3
   ```

   The client now fails with authentication errors (`KAFKA ERROR: ... SASL authentication error` or
   repeated broker disconnects) and no delivery lines appear.

6. Confirm that Entra ID access still works:

   ```bash
   AUTH=oauth python producer.py 3
   ```

   > **Result:** The security finding is resolved. There are no shared keys in config
   > files, access is attributable to an identity, and permissions are managed with
   > built-in roles instead of per-topic ACLs you would have to maintain. Authorization
   > with OAuth is superior to SAS in both security and ease of use.

---

## Task 4: Partitions, Consumer Groups and Idempotency

All commands below use Entra ID authentication (`AUTH=oauth` is still exported in your shell).

### Keys decide partitions

1. Send 30 events and count how many events each customer key sent to each partition:

   ```bash
   cd ~/lab7
   python producer.py 30 | awk '{print $1, $2}' | sort | uniq -c
   ```

   Every key (for example `C-100`) appears against **exactly one** partition. Events with
   the same key always land in the same partition, so **their relative order is preserved** for that
   customer.

   > Ordering in Kafka and Event Hubs is guaranteed *per partition*, not across the
   > whole topic. Choose a key (customerId, orderId, deviceId) that matches the unit that
   > must stay in order.

### Scale out within a consumer group

2. Start one consumer in the `analytics` group in the background and look at its log:

   ```bash
   python consumer.py analytics 150 > ~/c1.log 2>&1 &
   sleep 25; cat ~/c1.log
   ```

   Consumer 1 owns all four partitions: `ASSIGNED partitions: [0, 1, 2, 3]`, followed by the
   events the `analytics` group has not read yet (the 3 sent at the end of Task 3 plus the 30 produced above).

3. Start a second consumer in the **same** group and wait for the group to rebalance:

   ```bash
   python consumer.py analytics 150 > ~/c2.log 2>&1 &
   sleep 45
   grep -E "ASSIGNED|REVOKED" ~/c1.log ~/c2.log
   ```

   Consumer 1 logs `REVOKED` and then receives a smaller assignment, for example `[0, 1]`,
   while consumer 2 receives the rest, for example `[2, 3]`. The group has split the partitions
   between its members. (Rebalancing can take up to about 45 seconds.)

4. Produce more events and see the work being shared:

   ```bash
   python producer.py 40 > /dev/null
   sleep 15
   echo "consumer 1: $(grep -c MSG ~/c1.log) total"; echo "consumer 2: $(grep -c MSG ~/c2.log) total"
   ```

   Both consumers process events, each only from its own partitions. This is how a consumer
   group scales horizontally, and why **partitions cap parallelism**: a fifth consumer in this
   group would sit idle.

5. Stop the background consumers:

   ```bash
   pkill -f consumer.py; sleep 5
   ```

### Independent groups and replay

6. Run the `fraud-detection` group:

   ```bash
   python consumer.py fraud-detection 30 | tail -n 3
   ```

   The group has never read before, so `auto.offset.reset=earliest` makes it read **every
   retained event** (all events from Tasks 2–4) even though `analytics` has already processed them.
   Note the `processed=` count on the last line.

7. Run it again:

   ```bash
   python consumer.py fraud-detection 30 | tail -n 3
   ```

   This time `processed=0`: the group has committed its offsets and resumes where it left off.

8. Now replay the full history with a brand-new group:

   ```bash
   python consumer.py audit-replay 30 | tail -n 1
   ```

   The `processed=` count equals the first `fraud-detection` run. Because Event Hubs
   **retains** events for 24 hours regardless of whether they were read, a new consumer
   group, or an existing one with reset offsets, can replay history. This is a key
   difference from a queue, where a message is gone once it is completed.

### At-least-once delivery and idempotent consumers

9. Send 5 events, then re-send the last 3 with the same `eventId`, simulating an
   application-level retry. Then read with the `analytics` group:

   ```bash
   python producer.py 5 dup > /dev/null
   python consumer.py analytics 30 | grep -E "DUPLICATE|DONE"
   ```

   The consumer reports `DUPLICATE skipped` for 3 events and ends with
   `processed=5 duplicates=3`. (If a few events from step 4 were not committed before you
   stopped the consumers, `processed` can be slightly higher, but `duplicates=3` stays the same.)

   > **Why this matters.**
   > - `enable.idempotence=true` on the producer stops the **broker** from storing duplicates caused by *network
   >   retries inside one producer session*.
   > - It cannot recognise an application that deliberately (or after a crash) sends the same business event twice.
   > - Delivery is also at least once towards consumers, for example when a consumer crashes
   >   after processing but before committing.
   >
   > So consumers must be **idempotent**: de-duplicate on a business key such as `eventId`.
   > This lab keeps the IDs in memory. In production, keep them in a durable store, such
   > as the Day 6 `OrderAudit` table or a database unique constraint.

---

## Task 5: Multi-Protocol Access and Capture

Event Hubs exposes **Kafka, AMQP and HTTPS** concurrently over the same data. You can write with one protocol and read with
another. This matters for the analytics team, who prefer native Azure tooling.

### Read Kafka-produced events with the native Azure SDK (AMQP)

1. Create an AMQP reader that uses the `azure-eventhub` SDK, not a Kafka library:

   ```bash
   cd ~/lab7
   cat <<'EOF' > amqp_reader.py
   import os, threading
   from collections import Counter
   from azure.eventhub import EventHubConsumerClient
   from azure.identity import DefaultAzureCredential

   counts = Counter()
   client = EventHubConsumerClient(
       fully_qualified_namespace=os.environ["EH_NAMESPACE"],
       eventhub_name="orders-stream",
       consumer_group="$Default",
       credential=DefaultAzureCredential(),
   )


   def on_event(partition_context, event):
       counts[partition_context.partition_id] += 1
       if sum(counts.values()) <= 5:
           print(f"partition={partition_context.partition_id} body={event.body_as_str()}")


   # Stop after 20 seconds
   threading.Timer(20, client.close).start()
   with client:
       client.receive(on_event=on_event, starting_position="-1")   # -1 = start of the stream

   print("AMQP events read per partition:", dict(counts))
   EOF

   python producer.py 10 > /dev/null
   python amqp_reader.py
   ```

   The first five JSON events are printed, followed by the count per partition. The events were
   written with the **Kafka protocol** and read with **AMQP**, with no conversion step.

   > Azure Stream Analytics, Azure Functions and other Azure integrations use the native
   > Event Hubs interface, so the analytics team can consume the same stream
   > without any Kafka tooling. The `$Default` consumer group exists on every event hub.

### Archive the stream with Event Hubs Capture

2. Create a storage account and container for the archive:

   ```bash
   SA="stlab7yourname"      # 3-24 lowercase letters and digits, globally unique
   az storage account create -g $RG -n $SA -l australiaeast --sku Standard_LRS
   az storage container create --account-name $SA -n orders-capture --auth-mode key
   ```

3. Enable Capture on the event hub:

   ```bash
   az eventhubs eventhub update -g $RG --namespace-name $NS -n $HUB \
     --enable-capture true --capture-interval 60 --capture-size-limit 10485760 \
     --destination-name EventHubArchive.AzureBlockBlob \
     --storage-account $SA --blob-container orders-capture
   ```

   > **Capture** automatically writes the stream to Blob Storage in Avro format on a time
   > or size window (here 60 seconds or 10 MB), and works with events written over Kafka.
   > It is a cheap, long-term archive for the compliance team and for batch analytics. For
   > production, prefer a managed identity for the capture destination.

4. Produce events, wait for the capture window, and list the blobs:

   ```bash
   python producer.py 20 > /dev/null
   sleep 90
   az storage blob list --account-name $SA -c orders-capture --auth-mode key \
     --query "[].{name:name, bytes:properties.contentLength}" -o table
   ```

   Avro files named `<namespace>/orders-stream/<partition>/<year>/...` appear. Files
   for partitions that received no events during a window may be created empty. If the
   list is empty, wait another minute and run the command again.

### Observe throughput

5. In the portal, open the `ehns-lab7-yourname` namespace → **Metrics**. Add the metrics
   **Incoming Messages**, **Outgoing Messages** and **Captured Messages** (Sum), and set the time range to
   **Last 30 minutes**.

   You should see the spikes from each producer run. Incoming and outgoing traffic are
   different because each consumer group reads the stream separately, and the captured
   messages appear after the capture window closes. This is the same Azure Monitor
   data you used in Day 6, and you can route it to the same Log Analytics workspace using
   diagnostic settings.

---

## Task 6: Consume the Stream with an Azure Function

The fraud team does not want to run and scale consumer processes. In this task you replace
the hand-run `consumer.py` with an **Azure Function** that uses the native **Event Hubs trigger**.
The platform runs it, scales it, and checkpoints its progress, and it authenticates with a
**managed identity**. This is also required, because you disabled SAS authentication in Task 3.

The function reads batches of order events and applies two simple fraud-check rules:

| Rule | Condition | Log message |
| --- | --- | --- |
| Bulk order | `qty` is `3` | `FRAUD-CHECK bulk order` |
| Velocity | A customer has more than `5` orders in one batch | `FRAUD-CHECK velocity` |

### Create the Function App

1. Set variables, and create a consumer group for the function. Replace `yourname` with your own value.

   ```bash
   cd ~/lab7
   APP="func-lab7-yourname"           # globally unique
   FSA="stfunclab7yourname"           # 3-24 lowercase letters and digits, globally unique
   NS_ID=$(az eventhubs namespace show -g $RG -n $NS --query id -o tsv)

   az eventhubs eventhub consumer-group create -g $RG --namespace-name $NS \
     --eventhub-name $HUB --name fraud-function
   ```

   > **Why a new consumer group?** The Functions host stores its checkpoints in Blob Storage, whereas
   > the Kafka consumers in Task 4 commit offsets to Event Hubs. Using a separate group keeps the
   > two mechanisms from interfering with each other.

2. Create a dedicated storage account and a Python function app on the **Flex Consumption** plan:

   ```bash
   az storage account create -g $RG -n $FSA -l australiaeast --sku Standard_LRS --allow-blob-public-access false

   az functionapp create -g $RG -n $APP --storage-account $FSA \
     --flexconsumption-location australiaeast --runtime python --runtime-version 3.11
   ```

   > **Region check:** Flex Consumption is not available in every region. If the command reports that
   > the region is unsupported, list the supported regions with
   > `az functionapp list-flexconsumption-locations -o table` and use the same region for the new resources.
   >
   > **Why a separate storage account?** The Functions host stores code, state and checkpoints in
   > this account. Keep it separate from the Capture archive, and limit who can access it.

### Give the function a managed identity

3. Enable a system-assigned managed identity, grant it read access to the event stream,
   and point the trigger's connection at the namespace:

   ```bash
   PRINCIPAL_ID=$(az functionapp identity assign -g $RG -n $APP --query principalId -o tsv)

   az role assignment create --assignee-object-id $PRINCIPAL_ID --assignee-principal-type ServicePrincipal \
     --role "Azure Event Hubs Data Receiver" --scope $NS_ID

   az functionapp config appsettings set -g $RG -n $APP \
     --settings "EventHubConnection__fullyQualifiedNamespace=$EH_NAMESPACE"
   ```

   > **Identity-based connection.** The app setting is named `<CONNECTION_NAME>__fullyQualifiedNamespace`.
   > The trigger's `connection="EventHubConnection"` refers to this prefix, so the Functions runtime
   > obtains Entra ID tokens for the identity and **no connection string or key exists anywhere**.
   > Only the **Data Receiver** role is assigned, which is the least privilege a trigger needs.

### Create and deploy the function

4. Create the function project:

   ```bash
   mkdir -p ~/lab7/fraud-func && cd ~/lab7/fraud-func

   cat <<'EOF' > requirements.txt
   azure-functions
   EOF

   cat <<'EOF' > host.json
   {
     "version": "2.0",
     "extensionBundle": {
       "id": "Microsoft.Azure.Functions.ExtensionBundle",
       "version": "[4.*, 5.0.0)"
     }
   }
   EOF

   cat <<'EOF' > function_app.py
   import json
   import logging
   from collections import Counter

   import azure.functions as func

   app = func.FunctionApp()


   @app.function_name(name="FraudDetector")
   @app.event_hub_message_trigger(
       arg_name="events",
       event_hub_name="orders-stream",
       connection="EventHubConnection",
       consumer_group="fraud-function",
       cardinality="many",            # receive events in batches
   )
   def fraud_detector(events: list[func.EventHubEvent]):
       per_customer = Counter()
       for event in events:
           order = json.loads(event.get_body().decode("utf-8"))
           per_customer[order["customerId"]] += 1
           if order["qty"] == 3:
               logging.warning("FRAUD-CHECK bulk order eventId=%s customer=%s sku=%s",
                               order["eventId"][:8], order["customerId"], order["sku"])

       for customer, count in per_customer.items():
           if count > 5:
               logging.warning("FRAUD-CHECK velocity customer=%s orders_in_batch=%d", customer, count)

       logging.info("BATCH processed events=%d", len(events))
   EOF
   ```

   > | Part | Purpose |
   > | --- | --- |
   > | `event_hub_message_trigger` | Fires when events arrive. No polling loop, group join or offset code is needed. |
   > | `consumer_group="fraud-function"` | The group this function reads as. |
   > | `cardinality="many"` | Delivers a batch (a list) per invocation, which is more efficient at high volume. |
   > | `connection="EventHubConnection"` | Prefix of the identity-based app settings from Step 3. |

5. Package and deploy the project with a remote build (recommended for Python so that
   dependencies are built for Linux):

   ```bash
   rm -f ~/lab7/fraud-func.zip && zip -r ~/lab7/fraud-func.zip . > /dev/null

   az functionapp deployment source config-zip -g $RG -n $APP \
     --src ~/lab7/fraud-func.zip --build-remote true
   ```

   Deployment takes 2–3 minutes.

6. Confirm the function was registered:

   ```bash
   az functionapp function list -g $RG -n $APP --query "[].name" -o tsv
   ```

   The output should include `FraudDetector`. If it is empty, wait a minute and run the command again.

### Send traffic and observe the function

7. Produce a burst of orders with the Kafka producer from Task 2:

   ```bash
   cd ~/lab7
   export EH_NAMESPACE="$NS.servicebus.windows.net" AUTH=oauth
   python producer.py 60 > /dev/null
   ```

   > Kafka wrote these events, and the function reads them over the native Event Hubs interface (AMQP).
   > The new `fraud-function` group also starts at the beginning of the retained stream, so the
   > function first processes the history from earlier tasks, as the `audit-replay` group did in Task 4.

8. In the portal, open the `func-lab7-yourname` function app → **Application Insights**
   → **Logs**, switch to **KQL mode** if needed, and run:

   ```kusto
   traces
   | where message startswith "FRAUD-CHECK" or message startswith "BATCH processed"
   | order by timestamp desc
   | take 50
   ```

   You should see `FRAUD-CHECK bulk order` and `FRAUD-CHECK velocity` warnings, plus `BATCH processed events=N` lines.
   Telemetry can take 3–5 minutes to appear. If the query returns no rows, wait and rerun it.

   > The Application Insights resource is created automatically with the function app, in this resource
   > group. This is the same telemetry store and KQL you used in Day 6.

### Checkpoints and scale

9. List the checkpoint blobs that the Functions host wrote to its storage account:

   ```bash
   az storage blob list --account-name $FSA -c azure-webjobs-eventhub --auth-mode key \
     --query "[].name" -o tsv | head -n 8
   ```

   You should see blobs for the namespace, `orders-stream`, the `fraud-function` group and each of the
   4 partitions. These are the function's read positions, equivalent to the offsets that `consumer.py`
   committed to Event Hubs.

10. Restart the app and send a few new events to prove that it resumes where it left off:

    ```bash
    az functionapp restart -g $RG -n $APP
    sleep 30
    python producer.py 5 > /dev/null
    ```

    After a few minutes, rerun the KQL query from Step 8. The newest `BATCH processed` lines show only
    the 5 new events. The function did **not** re-read the whole stream.

    > **Scale-out.** On the Flex Consumption plan, the platform adds instances as events accumulate,
    > up to **one active consumer per partition** (4 here). This is the same "partitions cap
    > parallelism" rule you saw with `consumer.py` in Task 4, without any rebalancing code. Delivery is still
    > at least once, so a failed batch can be retried and the function should be idempotent. In
    > production, write results keyed by `eventId`.

---

## Task 7: Choose the Right Service

Kafka is excellent for high-throughput streams. It is **not** a one-stop messaging
platform. Kafka has no competing-consumer queue pattern, no server-evaluated
subscription rules, no per-message lifecycle tracking and no dead-letter queue. For these,
Azure offers Service Bus and Event Grid.

1. For each workload, decide which service fits best (**Event Hubs**, **Service Bus** or **Event Grid**),
   and write down why. Then compare with the answer key below.

   | # | Workload |
   | --- | --- |
   | 1 | Website clickstream: 50,000 events/second, read by analytics and fraud teams independently, replayable for 24 hours |
   | 2 | Customer order submissions that must be processed exactly once by a worker, with retries and a dead-letter queue (Day 6 `order-intake`) |
   | 3 | Notify invoice and warehouse services about a new order, each with its own filtering rules (Day 6 `order-notifications`) |
   | 4 | React when a blob lands in Storage, so that a function resizes the image |
   | 5 | Warehouse scanner telemetry from existing Kafka producers that must be archived to Blob cheaply |

   <details>
   <summary>Answer key</summary>

   | # | Best fit | Reason |
   | --- | --- | --- |
   | 1 | **Event Hubs** | High-throughput partitioned log. Consumer groups give independent readers, and retention allows replay. |
   | 2 | **Service Bus queue** | Competing consumers, peek-lock, max delivery count and dead-lettering. Kafka has none of these. |
   | 3 | **Service Bus topic + subscriptions** | Server-side filtering per subscription. Kafka subscribers can only filter by offset. |
   | 4 | **Event Grid** | Reactive, discrete Azure resource events pushed to a handler, with no polling. |
   | 5 | **Event Hubs (Kafka endpoint) + Capture** | Existing Kafka clients with only configuration changes, plus native Capture to Blob. |

   </details>

   > **Going further (not part of this lab):** Kafka compression (gzip), Kafka Streams and
   > Kafka transactions on Event Hubs need the **Premium** or **Dedicated** tier, and Streams and
   > transactions are in preview. If you want Kafka Streams-style analytics without it, Event Hubs also
   > integrates with Azure Stream Analytics, Azure Databricks, Apache Spark and Apache Flink. If you need
   > Kafka features that Event Hubs does not offer, a native Kafka cluster can run on Azure HDInsight.

---

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| `Authorization failed` or `Topic authorization failed` with OAuth | The role assignment has not propagated yet (wait 1–3 minutes), or the roles were assigned at a different scope than the namespace. |
| `SASL authentication error` with SAS in Task 2 | The `EH_CONNECTION_STRING` variable is empty or truncated. Echo its length, and rerun the Task 2 Step 3 command. Cloud Shell resets variables when a session restarts. |
| Consumer exits immediately with `processed=0` | The group already committed all offsets. Produce more events, or use a different consumer group. |
| Consumer shows no `ASSIGNED` line | The group join or rebalance is still in progress. Wait up to 45 seconds. |
| `ModuleNotFoundError` | Rerun the `pip install` command from Task 2, Step 2. |
| `$ConnectionString` appears empty | Use the heredoc with `'EOF'` in single quotes, as shown, so that the shell does not expand `$ConnectionString` or `$Default`. |
| No blobs in the capture container | Wait for the 60-second capture window to close and rerun the list command. |
| Function deployment fails or `FraudDetector` is not listed | Make sure `--build-remote true` was used, that the zip was created from **inside** `fraud-func` (so `function_app.py` is at the root of the zip), and that the region supports Flex Consumption. |
| Function runs but logs `Unauthorized` or no events are processed | The managed identity needs the **Azure Event Hubs Data Receiver** role on the namespace (allow 1–3 minutes to propagate), and the `EventHubConnection__fullyQualifiedNamespace` setting must be exactly `<namespace>.servicebus.windows.net`. |
| `traces` query returns no rows | Application Insights ingestion takes 3–5 minutes. Check **Functions → FraudDetector → Invocations** in the portal for earlier confirmation. |
| `az storage blob list` shows no checkpoint blobs | The function has not processed a batch yet. Send more events and wait a minute. |
| Variables such as `$RG`, `$NS` or `$HUB` are empty | Cloud Shell restarted. Set them again with the Task 2, Step 1 commands. |

---

## Cleanup

**Note:** Event Hubs Standard (throughput unit), the storage accounts, the Function App and Application Insights incur charges while they exist. Delete resources promptly after the lab.

1. Stop any leftover background consumers and clear the secrets from your shell:

   ```bash
   pkill -f consumer.py
   unset EH_CONNECTION_STRING AUTH
   ```

2. In the portal, navigate to **RG-Lab7**.

3. Select **Delete resource group**.

4. Copy and paste the resource group name to confirm deletion.

5. Select **Delete**.

---

## Key Takeaways

- **Event Hubs speaks the Kafka protocol; it is not a Kafka cluster.** Kafka 1.0+ clients connect
  to one stable endpoint (`<namespace>.servicebus.windows.net:9093`) with a configuration
  change only. There are no brokers, disks or networks to manage and monitor. Concepts map directly: cluster →
  namespace, topic → event hub, with partitions, consumer groups and offsets unchanged.

- **Prefer Microsoft Entra ID over SAS.** `OAUTHBEARER` with Azure RBAC removes shared
  secrets, ties access to an identity, and replaces ACL maintenance with built-in roles. Disabling
  local authentication enforces this for every client. Remember that Owner does not grant
  data-plane access.

- **Partitions provide both ordering and parallelism.** Order is guaranteed per partition, so the
  message key determines which events stay in order. The number of partitions is the ceiling for
  parallel consumers in one group.

- **Consumer groups give independent views of the same stream.** Each group tracks its own offsets, so
  analytics, fraud and audit systems can read, and replay, the retained events without affecting each other.

- **Delivery is at least once, so consumers must be idempotent.** Idempotent producers prevent
  broker-side duplicates from retries, but only a de-duplicating consumer protects against
  application-level resends and crash-before-commit scenarios.

- **One stream, many protocols.** Events written with Kafka can be read over AMQP with the
  native Azure SDK and archived with Capture. Teams are not forced into Kafka tooling.

- **Serverless consumers remove consumer plumbing.** An Azure Function with an Event Hubs trigger
  and a managed identity needs no group-join code, no offset handling and no secrets. The platform
  checkpoints progress in storage and scales out to at most one instance per partition, but delivery
  is still at least once, so handlers must stay idempotent.

- **Use the right messaging service for each job.** Event Hubs is for high-volume streams
  and replay. Service Bus is for queues, topics, filtering and dead-lettering. Event Grid is for reactive
  discrete events. Most platforms, including this retail platform, use all three.
