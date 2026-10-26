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
- Build a no-code Azure Stream Analytics job that reads the stream with a managed identity
  and applies simple fraud-detection rules
- Decide which workloads belong on Event Hubs, Service Bus or Event Grid

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab7**.
  Note: You will be assigned your own unique resource group name in the sandbox
  environment to avoid conflicts with other participants. Create a resource group in
  the same region you will deploy resources to (e.g., Australia East). Name the
  resource group RG-Lab7-yourname. Note the resource group name for use in all
  lab tasks.
- Owner rights at the resource group level (required to create role assignments in Tasks 3 and 6)
- Azure Cloud Shell (**Bash**, browser-based) — no local tooling installation required
- Basic familiarity with Day 6 (Service Bus and Event Grid), which is referenced in Task 7

## Estimated Timing: 75 minutes

- Task 1: Provision Kafka-enabled Event Hubs
- Task 2: Produce and consume with SAS
- Task 3: Switch to Microsoft Entra ID (OAuth 2.0)
- Task 4: Partitions, consumer groups and idempotency
- Task 5: Multi-protocol access and Capture
- Task 6: Process the stream with Azure Stream Analytics (no code)
- Task 7: Choose the right service, then clean up


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
│         ├── Kafka groups (auto-created, offsets stored by the service):
│         │     analytics, fraud-detection, audit-replay
│         ├── Event Hubs consumer group: $Default       (AMQP reader)
│         ├── Event Hubs consumer group: (created by the Stream Analytics job)
│         └── Capture ──► Blob container: orders-capture (Avro files)
│
├── Storage Account (stlab7yourname)                  [Capture archive]
│
├── Stream Analytics job (asa-fraud-lab7-yourname)    [No-code editor]
│   ├── Managed identity ──► Event Hubs (input) and Storage Blob Data Contributor (output)
│   ├── Filter: qty = 3                         ──► fraud-alerts/bulk-orders
│   └── Group by customerId (1 min), count > 5  ──► fraud-alerts/velocity
│
├── Storage Account (stfraudlab7yourname)             [ADLS Gen2, fraud alerts]
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
- Task 6: Build a no-code Stream Analytics job that reads the stream with a managed identity
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

### Note the Kafka endpoint

6. On the namespace **Overview** pane, note the **Host name**
   (`ehns-lab7-yourname.servicebus.windows.net`). This, with port `9093`, is your Kafka
   `bootstrap.servers` address. Unlike a Kafka cluster, which needs access to every broker,
   Event Hubs exposes a **single stable endpoint** for all partitions.

### Consumer groups

You do not create consumer groups for the Kafka clients in this lab.

> **Kafka consumer groups are not Event Hubs consumer groups.** Kafka consumer groups on Event Hubs are
> **auto-created** the first time a client uses a `group.id`, store their offsets in the Event Hubs
> service, and don't appear in the Azure portal. Event Hubs consumer groups are created explicitly
> (portal, CLI or ARM) and are used by AMQP consumers such as the Stream Analytics job in Task 6,
> which keep their offsets in external storage. Kafka groups span the namespace, so choose group names carefully.
>
> Each Kafka group tracks its own offsets, so each group reads the **whole** event stream
> independently. In Task 4, the `analytics`, `fraud-detection` and `audit-replay` groups are created
> by running the consumer with that `group.id`. This is how the analytics and fraud teams can process
> every event without affecting each other.

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
   > Note: `pip` prints `dependency resolver ... ansible-core ... requires packaging`. This is a harmless warning about software preinstalled in Cloud Shell. Confirm the install with `python -c "import confluent_kafka, azure.identity, azure.eventhub; print('OK')"` and continue.
   
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
   from azure.identity import AzureCliCredential

   TOPIC = "orders-stream"


   def kafka_conf():
       ns = os.environ["EH_NAMESPACE"]
       conf = {
           "bootstrap.servers": f"{ns}:9093",     # the Event Hubs Kafka endpoint
           "security.protocol": "SASL_SSL",        # TLS is mandatory
           "socket.keepalive.enable": True,        # Event Hubs closes idle connections
           "metadata.max.age.ms": 180000,          # must be below 240000
           "error_cb": lambda e: print("KAFKA ERROR:", e, flush=True),
       }
       if os.environ.get("AUTH", "sas") == "oauth":
           cred = AzureCliCredential()   # Cloud Shell: uses your signed-in Azure CLI identity

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
   > `socket.keepalive.enable` and `metadata.max.age.ms` follow the Event Hubs guidance for librdkafka clients.
   > The producer also sets `request.timeout.ms=60000`. Other settings keep the client defaults.
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
   conf["request.timeout.ms"] = 60000   # librdkafka's 5000 ms default is too low for Event Hubs
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
           consumer.commit(message=msg, asynchronous=False)
   finally:
       consumer.close()
       print(f"DONE group={group} processed={processed} duplicates={duplicates}", flush=True)
   EOF
   ```

   > **At-least-once delivery.** Event Hubs guarantees events are delivered **at least once**,
   > so a consumer can receive the same event more than once. Committing offsets **after**
   > processing, and de-duplicating on `eventId`, is the idempotent-consumer pattern you
   > will test in Task 4.

### What the scripts do

| Script | What it does |
| --- | --- |
| `kafka_common.py` | Builds the Kafka connection settings. `AUTH=sas` (default) or `AUTH=oauth` selects the authentication method, so the other scripts never change. |
| `producer.py` | Sends JSON order events (`eventId`, `customerId`, `sku`, `qty`, `ts`) keyed by `customerId`, so each customer's events stay in one partition and in order. Idempotence stops broker-side duplicates from retries. The `dup` argument re-sends the last 3 events to simulate an application retry. |
| `consumer.py` | Reads as the consumer group you pass in (auto-created by Kafka). A new group starts at the oldest event, and offsets are committed only after processing. It skips events whose `eventId` it has already seen, prints partition assignments, and ends with a `DONE` summary. |

### Send and receive events

7. Send 20 events:

   ```bash
   python producer.py 20
   ```

   You should see 20 lines such as `key=C-300 partition=2 offset=0`, spread across partitions 0–3.

   > **You may also see a warning at the top** such as
   > `GETPID ... Failed to acquire idempotence PID from broker ... retrying`. This is a harmless
   > startup message. The idempotent producer retries and succeeds. Only a `FAILED:` line is an error.

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

10. In the Portal, open the `orders-stream` event hub → **Data Explorer**. Select **View events** to see the incoming messages.

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

3. Go to the Azure portal → your Event Hubs namespace → **Access control (IAM)** to verify that your role assignments are in place.

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

   > **How it works.** `consumer.py` and `producer.py` call `AzureCliCredential`, which in
   > Cloud Shell uses your signed-in identity (no `az login` needed), and request a token for
   > `https://<namespace>.servicebus.windows.net/.default`. `DefaultAzureCredential` is not used here
   > because in Cloud Shell its managed identity step fails with `AudienceNotSupported`.
   > On an Azure VM, App Service or AKS pod, you would use `ManagedIdentityCredential` (or
   > `DefaultAzureCredential`) instead, so **no secret exists anywhere**.

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
   from azure.identity import AzureCliCredential

   counts = Counter()
   client = EventHubConsumerClient(
       fully_qualified_namespace=os.environ["EH_NAMESPACE"],
       eventhub_name="orders-stream",
       consumer_group="$Default",
       credential=AzureCliCredential(),
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

   > Azure Stream Analytics (Task 6), Azure Functions and other Azure integrations use the native
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

## Task 6: Process the Stream with Azure Stream Analytics (No Code)

The fraud team does not want to write, run or scale consumer processes. In this task you replace
the hand-run `consumer.py` with an **Azure Stream Analytics** job that you build in the Event Hubs
portal with the **no-code editor**. The job reads the stream over the native Event Hubs interface
(AMQP) as its own consumer group, applies the fraud rules, and writes alerts to Azure Data Lake
Storage Gen2. It authenticates with a **managed identity**, which is also required because you
disabled SAS authentication in Task 3.

The job applies two simple fraud-check rules:

| Rule | Condition | Output path |
| --- | --- | --- |
| Bulk order | `qty` equals `3` | `bulk-orders/...` |
| Velocity | A customer has more than `5` orders in a 1-minute window | `velocity/...` |

### Create the alert storage

1. Create a storage account with a hierarchical namespace (Data Lake Storage Gen2) and a container
   for the alerts. Replace `yourname` with your own value.

   ```bash
   cd ~/lab7
   FRAUD_SA="stfraudlab7yourname"     # 3-24 lowercase letters and digits, globally unique

   az storage account create -g $RG -n $FRAUD_SA -l australiaeast --sku Standard_LRS \
     --enable-hierarchical-namespace true --allow-blob-public-access false
   az storage container create --account-name $FRAUD_SA -n fraud-alerts --auth-mode key
   ```

   > The no-code editor writes to **Azure Data Lake Storage Gen2**. Keep this account separate from
   > the Capture archive from Task 5.

### Build the job in the Event Hubs portal

2. In the Azure portal, open the `ehns-lab7-yourname` namespace, and open the `orders-stream` event hub.
   Under **Features**, select **Process data**, and select **Start** on the **Filter and ingest to ADLS Gen2** card.
   Enter the job name `asa-fraud-lab7-yourname`, and select **Create**.

3. Select the **Event Hub** tile and configure the input:

   | Setting | Value |
   | --- | --- |
   | Consumer group | Keep the default **Create new** option |
   | Serialization | **JSON** |
   | Authentication method | **Managed Identity** (system-assigned) |

   Select **Connect**.

   > **Why Managed Identity?** SAS authentication is disabled on the namespace, so a connection-string
   > input would fail. When you select a managed identity, the no-code editor grants the job's identity
   > the **Azure Event Hubs Data Owner** role. This needs your Owner rights. In production, assign
   > only **Azure Event Hubs Data Receiver** to an input-only job.
   >
   > **A new Event Hubs consumer group.** Unlike Kafka groups, Event Hubs consumer groups are not
   > auto-created, so the editor creates one for the job. This keeps its read position independent of
   > `amqp_reader.py` on `$Default`.

4. The editor shows live data only when events are flowing. In Cloud Shell, send some events
   so that the fields (`eventId`, `customerId`, `sku`, `qty`, `ts`) appear in **Data preview**:

   ```bash
   cd ~/lab7
   export EH_NAMESPACE="$NS.servicebus.windows.net" AUTH=oauth
   for i in 1 2 3; do python producer.py 10 > /dev/null; sleep 10; done
   ```

   If the fields do not appear, select **Autodetect fields**. Confirm that `qty` has the type **Int**.

### Add the bulk-order rule

5. Select the **Filter** tile, and set **qty** **equals** `3`.

6. Select the **Azure Data Lake Storage Gen2** tile and configure the output:

   | Setting | Value |
   | --- | --- |
   | Storage account | `stfraudlab7yourname` |
   | Container | `fraud-alerts` |
   | Authentication method | **Managed Identity** |
   | Path pattern | `bulk-orders/{date}/{time}` |
   | Serialization | **JSON** |

   Select **Connect**. The editor grants the job's identity the **Storage Blob Data Contributor**
   role on the storage account.

### Add the velocity rule

7. In the ribbon, under **Operations**, select **Group by** and connect it to the **Event Hub** tile. Configure it:

   | Setting | Value |
   | --- | --- |
   | Aggregation | **Count** on `eventId` |
   | Group by | `customerId` |
   | Time window | **Tumbling**, `1` minute |

8. Add a second **Filter** tile after **Group by**. Select the count field that **Group by** produces,
   and set it **greater than** `5`.

9. From the ribbon, add a second **ADLS Gen2** output and connect it to this **Filter** tile. Use the
   same storage account, container and managed identity, but set the **Path pattern** to
   `velocity/{date}/{time}`.

### Start the job

10. Optionally, select **Get static preview** on each tile to check the results. Then select **Save**, and select **Start**.
    Set **Output start time** to **Now**, keep the recommended number of **Streaming units**, and select **Start**.

11. In the event hub, open **Process data** → **Stream Analytics jobs**, and select **Refresh** until the
    job status is **Running**. This takes up to 2 minutes.

    > The job only outputs events that arrive **after** it starts (**Output start time** = **Now**).
    > To process history, you can choose a **Custom** start time. This is the Stream Analytics equivalent of
    > the `audit-replay` group in Task 4.

### Send traffic and verify

12. Produce a burst of orders with the Kafka producer from Task 2:

    ```bash
    cd ~/lab7
    python producer.py 60 > /dev/null
    ```

    > Kafka wrote these events, and Stream Analytics reads them over AMQP. The 60 events are spread over
    > 5 customers, so each customer has about 12 orders in the same 1-minute window. If the burst happens to
    > cross a window boundary, send another 60 events.

13. Wait 2–3 minutes (the output writes a batch after at most 1 minute), then list the alert files:

    ```bash
    az storage blob list --account-name $FRAUD_SA -c fraud-alerts --auth-mode key \
      --query "[?properties.contentLength > \`0\`].{name:name, bytes:properties.contentLength}" -o table
    ```

    You should see files under both `bulk-orders/` and `velocity/`.

14. Download one file from each path and look at the content:

    ```bash
    for P in bulk-orders velocity; do
      BLOB=$(az storage blob list --account-name $FRAUD_SA -c fraud-alerts --auth-mode key \
        --query "[?starts_with(name, '$P') && properties.contentLength > \`0\`].name | [0]" -o tsv)
      az storage blob download --account-name $FRAUD_SA -c fraud-alerts -n "$BLOB" \
        --auth-mode key --file ~/$P.json --no-progress > /dev/null
      echo "== $P"; head -n 3 ~/$P.json
    done
    ```

    Every `bulk-orders` record has `"qty":3`. Every `velocity` record shows a `customerId` with a count greater than `5`.

### Review the job

15. In **Process data** → **Stream Analytics jobs**, select **Open metrics** for the job. **Input events** and
    **Output events** rise after your producer run, and **Backlogged input events** stays near zero.

16. In the `orders-stream` event hub, open **Consumer groups**. The group created by the job is listed.
    The Kafka groups from Task 4 are not, because Kafka groups are not shown in the portal.

17. Stop the job so that it stops billing: in **Stream Analytics jobs**, select the job, and select **Stop**.

    > **Compared with `consumer.py`:** there is no group-join, offset or deployment code, and the platform
    > scales with streaming units. Delivery is still at least once. For the ADLS Gen2 output, the
    > preview **Exactly once delivery** write mode is available when you need it.

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
   | 2 | Customer order submissions that must be handled by exactly one competing worker, with retries and a dead-letter queue (Day 6 `order-intake`) |
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
| `DefaultAzureCredential failed ... AudienceNotSupported` with `AUTH=oauth` | `DefaultAzureCredential` can't get an Event Hubs token from the Cloud Shell managed identity endpoint. Use `AzureCliCredential` as shown in `kafka_common.py` and `amqp_reader.py`. You don't need `az login` in Cloud Shell. |
| Producer prints `GETPID ... Failed to acquire idempotence PID ... retrying` | A harmless startup warning. The idempotent producer retries and succeeds. Continue if the delivery lines appear and there is no `FAILED:` line. |
| `pip` prints `dependency resolver ... ansible-core ... requires packaging` | This is a harmless warning about software preinstalled in Cloud Shell. Confirm the install with `python -c "import confluent_kafka, azure.identity, azure.eventhub; print('OK')"` and continue. |
| `$ConnectionString` appears empty | Use the heredoc with `'EOF'` in single quotes, as shown, so that the shell does not expand `$ConnectionString` or `$Default`. |
| No blobs in the capture container | Wait for the 60-second capture window to close and rerun the list command. |
| The Process data card or job creation fails | Make sure the event hub is publicly accessible and the `Microsoft.StreamAnalytics` resource provider is registered for the subscription (**Subscriptions → Resource providers**). |
| Job input or output reports `Unauthorized` | The job's managed identity needs **Azure Event Hubs Data Owner** (or **Data Receiver**) on the event hub and **Storage Blob Data Contributor** on `stfraudlab7yourname`. Role assignments can take up to 8 minutes. A connection-string input fails because SAS is disabled. |
| The Data preview shows no fields | No events are flowing. Run the producer from Step 4, then select **Autodetect fields**. |
| No files in the `fraud-alerts` container | Make sure the job status is **Running** and that you sent events **after** it started. Output batches are written after at most 1 minute, so wait 2–3 minutes. Check **Runtime logs** in the editor. |
| Only `bulk-orders` files appear | No customer exceeded 5 orders in one 1-minute window, for example because the burst crossed a window boundary. Run `python producer.py 60` again. || Variables such as `$RG`, `$NS` or `$HUB` are empty | Cloud Shell restarted. Set them again with the Task 2, Step 1 commands. |

---

## Cleanup

**Note:** Event Hubs Standard (throughput unit), the storage accounts and the Stream Analytics job (billed per streaming unit while it runs) incur charges while they exist. Stop the job and delete resources promptly after the lab.

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

- **No-code stream processing removes consumer plumbing.** A Stream Analytics job built in the Event Hubs
  portal reads the stream over AMQP with a managed identity, needs no group-join code, no offset
  handling and no secrets, and scales with streaming units. Event Hubs consumer groups are created
  explicitly (the editor creates one for the job), unlike auto-created Kafka groups. Delivery is still at
  least once.

- **Use the right messaging service for each job.** Event Hubs is for high-volume streams
  and replay. Service Bus is for queues, topics, filtering and dead-lettering. Event Grid is for reactive
  discrete events. Most platforms, including this retail platform, use all three.
