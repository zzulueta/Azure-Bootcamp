---
lab:
  title: 'Lab Day 7: Streaming with Apache Kafka on Azure Event Hubs'
  module: 'Advanced Azure Bootcamp – Day 7'
---

# Lab Day 7 – Streaming with Apache Kafka on Azure Event Hubs

## Lab Introduction

In Day 6 you built an order pipeline with Event Grid, Service Bus and Logic Apps. Now the business
wants something bigger: a **stream** that can take tens of thousands of events per second, such as
clicks, cart activity and warehouse scanner data. Analytics and fraud systems will read from it.

The data engineering team already has applications that talk to **Apache Kafka**, a popular streaming
system. They do not want to rewrite them, and they do not want to run and fix Kafka servers any more.

**Azure Event Hubs** solves this. It is a service run by Microsoft that understands the Kafka protocol.
The Kafka applications connect to it by changing a few settings. There is no code rewrite, and there are
no servers to manage.

In this lab you will:

1. Create an Event Hubs namespace and an event hub (this is where the events live)
2. Connect a Kafka producer and consumer with a shared key (SAS)
3. Switch to Microsoft Entra ID sign-in and turn the shared keys off
4. Learn how partitions, consumer groups and duplicates behave
5. Read the same events with a non-Kafka tool, and save a copy to storage with Capture
6. Build a no-code Azure Stream Analytics job that finds possible fraud
7. Decide when to use Event Hubs, Service Bus or Event Grid

## Words you will see

| Word | Simple meaning |
| --- | --- |
| **Event** | One small message, for example "customer C-100 ordered a mug" |
| **Event hub** (Kafka calls it a **topic**) | A named place where events are stored in order |
| **Namespace** (Kafka calls it a **cluster**) | The container that holds your event hubs and gives you the address to connect to |
| **Partition** | One lane of an event hub. An event hub is split into several lanes so many readers can work at once |
| **Offset** | The position number of an event inside a partition (0, 1, 2…) |
| **Producer** | A program that **sends** events |
| **Consumer** | A program that **reads** events |
| **Consumer group** | A reader's name. Each name keeps its own bookmark, so it reads the stream on its own |
| **Key** | A label on an event, such as the customer ID. Events with the same key go to the same partition |
| **SAS** | Shared Access Signature. A shared secret string that works like a password |
| **Microsoft Entra ID** | Microsoft's sign-in service. You prove who you are instead of sharing a password |
| **Azure RBAC** | Roles that decide what a signed-in identity is allowed to do |
| **AMQP** | Another protocol Event Hubs understands. Azure's own tools use it |
| **Capture** | An Event Hubs feature that saves the stream to storage as files |

## Pre-requisites

- An Azure subscription
- A resource group named **RG-Lab7**.
  Note: You will be assigned your own unique resource group name in the sandbox environment to avoid
  conflicts with other participants. Create a resource group in the same region you will deploy resources
  to (for example, Australia East). Name it `RG-Lab7-yourname`. Use this name in all lab tasks.
- Owner rights on the resource group (needed to create role assignments in Tasks 3 and 6)
- Azure Cloud Shell (**Bash**, in the browser). You do not need to install anything
- Basic knowledge of Day 6 (Service Bus and Event Grid), which comes back in Task 7

## Estimated Timing: 90 minutes

| Task | What you do |
| --- | --- |
| 1 | Create the Event Hubs namespace and event hub |
| 2 | Send and read events with a shared key (SAS) |
| 3 | Switch to Microsoft Entra ID and turn SAS off |
| 4 | Learn partitions, consumer groups and duplicates |
| 5 | Read with AMQP and archive with Capture |
| 6 | Find possible fraud with Stream Analytics (no code) |
| 7 | Choose the right messaging service, then clean up |

## Lab Scenario

The retail platform from Days 5 and 6 is getting ready for its biggest flash sale. Service Bus handles
orders well, but the business now also wants to collect **clickstream, cart activity and warehouse-scanner
data** and send it to analytics and fraud-detection systems.

Three things shape the solution:

1. **Existing Kafka investment.** The data engineering team has working Kafka applications. They will not
   rewrite them, but they are tired of patching and rebalancing Kafka servers.
2. **Security finding.** The Kafka clients use shared keys saved in config files. Security wants sign-in by
   identity, with no shared secrets.
3. **Many readers.** The analytics team and the fraud team must each read every event on their own. The
   analytics team prefers Azure tools over Kafka tools, and the compliance team wants a cheap archive.

> **Is this a real Kafka cluster?** No. Event Hubs is a fully managed Azure service with a Kafka endpoint.
> You do not set up or manage any Kafka servers. It speaks the Kafka protocol (Kafka 1.0 and later) on
> port 9093. The Python code in this lab uses the standard `confluent-kafka` library, and it connects
> to Event Hubs just as it would connect to Kafka.

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
      ├── producer.py   (Kafka protocol, SAS or Entra ID)
      ├── consumer.py   (Kafka protocol, consumer groups, skips duplicates)
      └── amqp_reader.py (native azure-eventhub SDK, AMQP protocol)
```

## If Cloud Shell restarts

Cloud Shell forgets your variables when the session restarts. If a command fails because `$RG`, `$NS` or
`$EH_NAMESPACE` is empty, run this block again. Replace `yourname` with your own value:

```bash
cd ~/lab7
RG="RG-Lab7-yourname"
NS="ehns-lab7-yourname"
HUB="orders-stream"
export EH_NAMESPACE="$NS.servicebus.windows.net"
export AUTH=oauth          # use this only after Task 3
```

---

## Task 1: Create the Event Hubs namespace and event hub

**Goal:** create the place where your events will live.

Kafka and Event Hubs work in a very similar way. Both are **logs split into partitions**. The reader
decides which part of the stored log to read. Here is how the names match:

| Apache Kafka | Azure Event Hubs |
| --- | --- |
| Cluster | Namespace |
| Topic | Event hub |
| Partition | Partition |
| Consumer group | Consumer group |
| Offset | Offset |

> **There is no Kafka switch to turn on.** The Kafka endpoint is on by default for the **Standard, Premium
> and Dedicated** tiers. The **Basic** tier does not support Kafka.

### Create the namespace

1. In the Azure portal, search for and select **Event Hubs**.

2. Select **+ Create** and fill in the **Basics** tab:

   | Setting | Value |
   | --- | --- |
   | Resource group | **RG-Lab7** (use your own name) |
   | Namespace name | `ehns-lab7-yourname` (must be unique worldwide) |
   | Location | **Australia East** |
   | Pricing tier | **Standard** |
   | Throughput Units | `1` |
   | Enable Auto-Inflate | **Unchecked** |

   > **Why Standard?** It is the cheapest tier that supports Kafka. Some Kafka extras (gzip compression,
   > Kafka Streams, Kafka transactions) need Premium or Dedicated, so this lab does not use them.
   >
   > **How do you scale?** There are no servers to size. You buy capacity in *throughput units* (Standard) or
   > *processing units* (Premium).

3. Select **Review + Create**, then **Create**. It takes 1–2 minutes.

### Create the event hub

4. Open your namespace. In the left menu, under **Entities**, select **Event Hubs**.

5. Select **+ Event Hub** and fill in:

   | Setting | Value |
   | --- | --- |
   | Name | `orders-stream` |
   | Partition count | `4` |
   | Cleanup policy | **Delete** |
   | Retention time (hrs) | `24` |

   Select **Review + create**, then **Create**.

   > **What does this mean?**
   > - An **event hub is a Kafka topic**. Kafka programs see it as a topic named `orders-stream`. You create it
   >   in Azure, not with Kafka tools.
   > - **4 partitions** means 4 lanes. Events with the same key always use the same lane, so their order is kept.
   > - **Retention 24 hours** means Event Hubs keeps every event for a day, even after it has been read.

### Note the Kafka address

6. On the namespace **Overview** page, find the **Host name**: `ehns-lab7-yourname.servicebus.windows.net`.
   Add `:9093` to the end and you have the Kafka `bootstrap.servers` address. Unlike a real Kafka cluster,
   there is only **one address** for all partitions.

### About consumer groups

You do not create any consumer groups for the Kafka programs in this lab.

> **Kafka groups and Event Hubs groups are not the same thing.**
> - **Kafka consumer groups** are created automatically the first time a program uses a group name. They
>   keep their bookmarks inside Event Hubs, and they do **not** show in the Azure portal.
> - **Event Hubs consumer groups** are created by you (portal, CLI or template). Azure tools that use AMQP,
>   such as Stream Analytics in Task 6, use these.
>
> Each group has its own bookmark, so each group reads the whole stream on its own. This is how the analytics
> team and the fraud team can read every event without affecting each other.

**Check your work:** you have a namespace called `ehns-lab7-yourname` with an event hub called `orders-stream`
that has 4 partitions.

---

## Task 2: Send and read events with SAS

**Goal:** connect a normal Kafka program to Event Hubs, and see that only a few settings change.

The quickest way to connect is a **Shared Access Signature (SAS)** connection string. It works like a
password. Kafka's `PLAIN` sign-in method sends it over an encrypted (TLS) connection. In Task 3 you will
replace it with something safer.

### Prepare Cloud Shell

1. Open Cloud Shell (**Bash**) and set your variables. Replace `yourname` with your own value.

   ```bash
   RG="RG-Lab7-yourname"
   NS="ehns-lab7-yourname"
   HUB="orders-stream"
   export EH_NAMESPACE="$NS.servicebus.windows.net"
   ```

2. Install the Python libraries:

   ```bash
   pip install confluent-kafka azure-identity azure-eventhub --quiet --user
   ```

   > `pip` may print `dependency resolver ... ansible-core ... requires packaging`. This is a harmless warning
   > about software that Cloud Shell already has. Check the install with:
   >
   > `python -c "import confluent_kafka, azure.identity, azure.eventhub; print('OK')"`

3. Create a SAS policy that can only **send and listen**, and read its connection string:

   ```bash
   az eventhubs namespace authorization-rule create \
     --resource-group $RG --namespace-name $NS \
     --name kafka-app --rights Send Listen

   export EH_CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
     --resource-group $RG --namespace-name $NS \
     --name kafka-app --query primaryConnectionString -o tsv)
   ```

   > **Why not use `RootManageSharedAccessKey`?** It can manage the whole namespace. A program that only sends
   > and reads should only have Send and Listen rights.
   >
   > Event Hubs Kafka accepts the namespace **connection string** as the password. It does not accept generated
   > SAS *tokens*.

### Create the Kafka programs

4. Create a folder and a shared settings file:

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

   > | Setting | What it does |
   > | --- | --- |
   > | `bootstrap.servers` | The address of Event Hubs. This is the main setting that points away from your own Kafka cluster |
   > | `security.protocol=SASL_SSL` | Required. All traffic is encrypted |
   > | `sasl.mechanism=PLAIN` + `$ConnectionString` | SAS sign-in. The user name is the exact text `$ConnectionString`. The password is your connection string |
   > | `sasl.mechanism=OAUTHBEARER` | Microsoft Entra ID sign-in, used in Task 3 |
   >
   > The `AUTH` variable switches between the two sign-in methods. The producer and consumer code does not change.

5. Create the producer (it **sends** events):

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

6. Create the consumer (it **reads** events):

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

### What the programs do

| Program | What it does |
| --- | --- |
| `kafka_common.py` | Builds the connection settings. `AUTH=sas` (the default) or `AUTH=oauth` picks the sign-in method |
| `producer.py` | Sends order events as JSON (`eventId`, `customerId`, `sku`, `qty`, `ts`). The key is `customerId`, so one customer's events stay in one partition and in order. The `dup` option sends the last 3 events a second time to copy an application retry |
| `consumer.py` | Reads as the group name you give it. A new group starts at the oldest event. It saves its bookmark (commits) only **after** it processes an event, and skips an event if it has already seen its `eventId`. It stops after the number of quiet seconds you give it, and prints a `DONE` summary |

> **Why does the consumer skip repeated events?** Event Hubs delivers events **at least once**. In rare cases
> the same event can arrive twice. A good consumer checks the `eventId` and ignores repeats. You will test this
> in Task 4.

### Send and read events

7. Send 20 events:

   ```bash
   python producer.py 20
   ```

   **What you should see:** 20 lines like `key=C-300 partition=2 offset=0`, spread over partitions 0–3.

   > You might also see a warning at the top: `GETPID ... Failed to acquire idempotence PID ... retrying`. It is a
   > harmless start-up message. The producer retries and succeeds. Only a line that starts with `FAILED:` is an
   > error.

8. Read them with the `analytics` consumer group:

   ```bash
   python consumer.py analytics 30
   ```

   The `30` means "stop after 30 seconds with no new events".

   **What you should see:** `ASSIGNED partitions: [0, 1, 2, 3]`, then 20 `MSG` lines, then
   `DONE group=analytics processed=20 duplicates=0`.

   > The `analytics` group did not exist before this command. Kafka created it the first time you used the name.

9. In the portal, open the `orders-stream` event hub and go to **Overview**. After 1–2 minutes the **Messages**
   charts show 20 incoming and 20 outgoing messages.

10. In the event hub, open **Data Explorer** and select **View events** to see the events.

> **What changed compared with a normal Kafka client?** Only `bootstrap.servers` and the three SASL lines. The
> producer and consumer logic is plain Kafka code.

**Check your work:** the producer sent 20 events and the `analytics` group read all 20.

---

## Task 3: Switch to Microsoft Entra ID

**Goal:** remove the shared key and sign in with your identity instead.

A SAS key is a shared secret. Anyone who has the string can use it, it has to be stored somewhere, and
changing it affects every program. (Also, if you regenerate a SAS key on the Kafka endpoint, connections that
are already open are **not** closed.)

With **Microsoft Entra ID**, a program proves **who it is**, and **Azure RBAC** roles decide **what it may do**.

### Give yourself the data roles

1. Give yourself the two data roles on the namespace:

   ```bash
   NS_ID=$(az eventhubs namespace show -g $RG -n $NS --query id -o tsv)
   ME=$(az ad signed-in-user show --query id -o tsv)

   az role assignment create --assignee-object-id $ME --assignee-principal-type User \
     --role "Azure Event Hubs Data Sender" --scope $NS_ID
   az role assignment create --assignee-object-id $ME --assignee-principal-type User \
     --role "Azure Event Hubs Data Receiver" --scope $NS_ID
   ```

   > **Owner is not enough.** Owner lets you manage Azure resources (the *management plane*). Sending and reading
   > events is a *data* action (the *data plane*), and it needs the Data Sender and Data Receiver roles.
   >
   > If `az ad signed-in-user show` is blocked in your sandbox, add both roles in the portal: namespace →
   > **Access control (IAM)** → **Add role assignment**.

2. Wait 1–2 minutes for the roles to take effect.

3. In the portal, open the namespace → **Access control (IAM)** and check that your two roles are listed.

### Run the Kafka programs with Entra ID

4. Sign in to the Azure CLI with a full login. The Python programs use this sign-in to get a token for
   Event Hubs. Cloud Shell's built-in sign-in cannot create tokens for Event Hubs, so you need this step.
   Follow the device-code instructions in your browser:

   ```bash
   az login --use-device-code
   az account show --query "{user:user.name, subscription:name}" -o table
   az account get-access-token --resource https://$EH_NAMESPACE --query expiresOn -o tsv
   ```

   **What you should see:** your user name and subscription, then a date and time from the last command. Make
   sure the subscription is the one that has your resource group.

5. Switch both programs to Entra ID:

   ```bash
   cd ~/lab7
   export AUTH=oauth
   python producer.py 10
   python consumer.py analytics 30
   ```

   **What you should see:** 10 delivery lines from the producer, and the consumer reads those 10 new events. If you
   see an authorization error, wait another minute and try again.

   > **How does it work?** The programs use `AzureCliCredential`, which uses the identity from `az login`. They ask
   > for a token for `https://<namespace>.servicebus.windows.net/.default` and send that token to Event Hubs. No
   > password or key is stored anywhere.
   >
   > Why not `DefaultAzureCredential`? In Cloud Shell its managed-identity step fails with `AudienceNotSupported`.
   > On an Azure VM, App Service or AKS you would use a managed identity (`ManagedIdentityCredential`) instead.

### Turn SAS off

6. Turn off key-based (local) authentication on the namespace:

   ```bash
   az eventhubs namespace update -g $RG -n $NS --disable-local-auth true
   ```

7. Wait about 1 minute, then try the old SAS sign-in:

   ```bash
   AUTH=sas timeout 40 python producer.py 3
   ```

   **What you should see:** it **fails**. You get `FAILED: KafkaError{code=TOPIC_AUTHORIZATION_FAILED ...}` and no
   `key=… partition=… offset=…` lines. This is the correct result, because SAS is now switched off.

8. Check that Entra ID still works:

   ```bash
   AUTH=oauth python producer.py 3
   ```

   **What you should see:** 3 delivery lines.

> **Result:** the security finding is fixed. No shared keys sit in config files, every action is tied to an
> identity, and access is controlled by built-in roles. Entra ID sign-in is safer and easier to manage than SAS.

**Check your work:** the SAS producer fails, and the Entra ID producer works.

---

## Task 4: Partitions, consumer groups and duplicates

**Goal:** learn four ideas by running the producer and consumer you already built.

| Part | Idea | What you will see |
| --- | --- | --- |
| 1 | Keys choose partitions | One customer's events always go to the same partition, so they stay in order |
| 2 | Consumers in the same group share work | Two consumers split the partitions between them |
| 3 | Different groups read independently | A new group reads all history. An old group continues from its bookmark |
| 4 | Duplicates happen | A consumer skips repeated events |

> **Picture it as a logbook with 4 pages (partitions).** Producers add lines at the end of a page. A **consumer
> group** is a reader's name, and each name has its own bookmark. Two readers with the **same** name share the pages.
> Readers with **different** names each read the whole logbook. Reading never erases anything, because Event Hubs
> keeps events for 24 hours.

**Before you start:** these commands use Entra ID sign-in. If Cloud Shell restarted, run the "If Cloud Shell
restarts" block near the top of this lab, and run `az login --use-device-code` if the CLI asks you to sign in.

### Part 1: Keys choose partitions

1. Send 30 events and count how many events each customer sent to each partition:

   ```bash
   cd ~/lab7
   python producer.py 30 | awk '{print $1, $2}' | sort | uniq -c
   ```

   **What you should see:** one row per customer, for example `8 key=C-100 partition=3`. Each customer shows up
   next to only **one** partition. The numbers add up to 30.

   **What it means:** Kafka uses the key to pick the partition, so the same customer always goes to the same
   partition. Inside a partition, events stay in the order they were sent. Two customers can share a partition,
   but one customer never splits across two.

   > Order is only guaranteed **inside one partition**, not across the whole event hub. Choose a key (customer ID,
   > order ID, device ID) that matches the thing that must stay in order.

### Part 2: Consumers in the same group share the work

**Goal:** run two consumers with the same group name and watch Kafka split the partitions between them.

2. Run this whole block. It starts consumer 1, waits, starts consumer 2, sends 40 events, shows how the work was
   shared, and stops both consumers. It takes about two minutes:

   ```bash
   pkill -f consumer.py 2>/dev/null; sleep 3

   # Consumer 1 starts alone
   python consumer.py analytics 300 > ~/c1.log 2>&1 &
   sleep 30
   echo "--- consumer 1 alone:"; grep -E "ASSIGNED" ~/c1.log; echo "events read: $(grep -c MSG ~/c1.log)"

   # Consumer 2 joins the same group, so the group rebalances
   python consumer.py analytics 300 > ~/c2.log 2>&1 &
   sleep 45
   echo "--- after consumer 2 joined:"; grep -E "ASSIGNED|REVOKED" ~/c1.log ~/c2.log

   # New events arrive and are shared
   python producer.py 40 > /dev/null
   sleep 15
   echo "--- totals:"
   echo "consumer 1: $(grep -c MSG ~/c1.log) events"; echo "consumer 2: $(grep -c MSG ~/c2.log) events"

   # Stop both. Wait so Event Hubs notices they left the group.
   pkill -f consumer.py; sleep 35
   ```

   **What you should see:**
   - **Consumer 1 alone:** `ASSIGNED partitions: [0, 1, 2, 3]` and about 33 events read (the 3 sent at the end of
     Task 3, plus the 30 from Part 1). It has all four partitions because nobody else is in the group.
   - **After consumer 2 joins:** consumer 1 prints `REVOKED`, then a smaller `ASSIGNED`, for example `[0, 1]`.
     Consumer 2 prints `ASSIGNED` with the rest, for example `[2, 3]`. This can take up to about 45 seconds.
   - **Totals:** both consumers are above zero. Consumer 1's total includes its first ~33 events, so the two
     totals add up to about 73 (33 + 40).

   **What it means:**
   - Both consumers use the group name `analytics`, so Kafka treats them as one team and divides the partitions.
     One partition is read by only one member of the group at a time.
   - `REVOKED` means consumer 1 gave up some partitions so consumer 2 could take them.
   - To handle more load, you add consumers. **But the number of partitions is the limit.** With 4 partitions, a
     fifth consumer in the group would have nothing to read.

   > If one consumer shows a total of 0, the group had not finished rebalancing. Run the block again. If a consumer
   > had already exited (it stops after 300 quiet seconds), the other consumer takes all partitions and nothing is
   > shared.

### Part 3: Different groups read independently

**Goal:** show that each group name has its own bookmark.

3. Run a brand-new group called `fraud-detection`:

   ```bash
   python consumer.py fraud-detection 30 | tail -n 3
   ```

   **What you should see:** the last line is `DONE group=fraud-detection processed=N duplicates=0`. Write down `N`. It
   should be about 100, because it counts every event you have sent so far (roughly 20 + 10 + 3 + 30 + 40, or more
   if you ran the producer extra times).

   **What it means:** `fraud-detection` is new, so it has no bookmark. The setting `auto.offset.reset=earliest`
   makes it start at the oldest event Event Hubs still keeps. It reads everything, even though `analytics` already
   read the same events. Groups do not affect each other.

4. Run the same group again:

   ```bash
   python consumer.py fraud-detection 30 | tail -n 3
   ```

   **What you should see:** `DONE group=fraud-detection processed=0 duplicates=0`.

   **What it means:** after the first run, the group saved its bookmark. When it starts again it continues from
   there, so there is nothing new to read.

5. Replay all the history with another brand-new group:

   ```bash
   python consumer.py audit-replay 30 | tail -n 1
   ```

   **What you should see:** `DONE group=audit-replay processed=N`, with the same `N` as the first `fraud-detection`
   run.

   **What it means:** a new group starts at the oldest event and reads the whole history again. Event Hubs keeps
   events for 24 hours whether or not they were read, so a new group (or a group whose bookmark was reset) can
   replay old events at any time. A queue is different: once a message is completed, it is gone.

### Part 4: Duplicates happen

**Goal:** see why a consumer must be able to skip an event it already handled.

6. Send 5 events, then send the last 3 again with the same `eventId`. This copies an application that retries after
   an error. Then read with the `analytics` group:

   ```bash
   python producer.py 5 dup > /dev/null
   python consumer.py analytics 30 | grep -E "DUPLICATE|DONE"
   ```

   **What you should see:** three `DUPLICATE skipped eventId=…` lines, then `DONE group=analytics processed=5
   duplicates=3`. (If a few events from Part 2 were not saved before you stopped the consumers, `processed` can be a
   little higher, but `duplicates=3` stays the same.)

   **What it means:** the producer sent 8 messages, but only 5 were different events. The last 3 were copies with
   the same `eventId`. The consumer remembers each `eventId` it has seen, so it processes each real event once and
   skips the copies.

   > **Why this matters:**
   > - `enable.idempotence=true` on the producer stops the **broker** from storing duplicates that come from network
   >   retries inside one producer session.
   > - It cannot spot an application that sends the same business event twice on purpose, or after a crash.
   > - Consumers can also see an event twice, for example if they crash after processing but before saving the
   >   bookmark.
   >
   > So consumers must be **idempotent**: processing the same event twice must give the same result. Do this by
   > de-duplicating on a business ID such as `eventId`. This lab keeps the IDs in memory. In production, keep them in
   > a durable store, such as the Day 6 `OrderAudit` table or a database unique constraint.

**Check your work:** you saw one customer stay in one partition, two consumers split the partitions, a new group
read the full history, and a consumer skip 3 duplicates.

---

## Task 5: Read with AMQP and archive with Capture

**Goal:** show that non-Kafka tools can read the same events, and save a copy of the stream to storage.

Event Hubs understands **Kafka, AMQP and HTTPS** on the same data. You can write with one protocol and read with
another. This is useful for the analytics team, who prefer Azure tools.

### Read Kafka events with the Azure SDK (AMQP)

1. Create an AMQP reader that uses the `azure-eventhub` SDK, not a Kafka library, and run it:

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

   **What you should see:** the first five JSON events, then a count per partition.

   **What it means:** the events were written with the **Kafka protocol** and read with **AMQP**, with no conversion.
   Azure Stream Analytics (Task 6) and other Azure services use this same native interface, so teams do not need
   Kafka tools. Every event hub has a `$Default` consumer group, which is the one this reader uses.

### Archive the stream with Capture

2. Create a storage account and a container for the archive:

   ```bash
   SA="stlab7yourname"      # 3-24 lowercase letters and digits, unique worldwide
   az storage account create -g $RG -n $SA -l australiaeast --sku Standard_LRS
   az storage container create --account-name $SA -n orders-capture --auth-mode key
   ```

3. Turn on Capture for the event hub:

   ```bash
   az eventhubs eventhub update -g $RG --namespace-name $NS -n $HUB \
     --enable-capture true --capture-interval 60 --capture-size-limit 10485760 \
     --destination-name EventHubArchive.AzureBlockBlob \
     --storage-account $SA --blob-container orders-capture
   ```

   > **What is Capture?** It saves the stream to Blob Storage as Avro files, every 60 seconds or every 10 MB,
   > whichever comes first. It works for events written over Kafka. It is a cheap long-term archive for the
   > compliance team and for batch analytics. It only saves events that arrive **after** you turn it on. In
   > production, use a managed identity for the capture destination.

4. Send events, wait for the capture window, and list the files:

   ```bash
   python producer.py 20 > /dev/null
   sleep 90
   az storage blob list --account-name $SA -c orders-capture --auth-mode key \
     --query "[].{name:name, bytes:properties.contentLength}" -o table
   ```

   **What you should see:** Avro files with names like `<namespace>/orders-stream/<partition>/<year>/...`. Some may be
   empty if a partition got no events in that window. If the list is empty, wait a minute and run the list command
   again.

### Watch the traffic

5. In the portal, open your namespace → **Metrics**. Add **Incoming Messages**, **Outgoing Messages** and **Captured
   Messages** (Sum), and set the time range to **Last 30 minutes**.

   **What you should see:** a spike for each producer run. Incoming and outgoing counts differ because each consumer
   group reads the stream separately, and captured messages show up after each capture window closes. You can send this
   same Azure Monitor data to the Log Analytics workspace you used in Day 6.

**Check your work:** the AMQP reader printed events, and files appear in the `orders-capture` container.

---

## Task 6: Find possible fraud with Stream Analytics (no code)

**Goal:** replace a hand-run consumer with a managed Azure service, without writing code.

The fraud team does not want to write, run or scale consumer programs. **Azure Stream Analytics** can read the event
hub and apply rules for you. You build the job by dragging boxes in the portal (the **no-code editor**). The job reads
over AMQP as its own consumer group, and it signs in with a **managed identity**. You need this because SAS is off.

The job applies two simple fraud rules:

| Rule | Condition | Where results go |
| --- | --- | --- |
| Bulk order | `qty` equals `3` | `bulk-orders/...` |
| Velocity | A customer has more than `5` orders in a 1-minute window | `velocity/...` |

### Create the storage for alerts

1. Create a Data Lake Storage Gen2 account (a storage account with a hierarchical namespace) and a container.
   Replace `yourname` with your own value.

   ```bash
   cd ~/lab7
   FRAUD_SA="stfraudlab7yourname"     # 3-24 lowercase letters and digits, unique worldwide

   az storage account create -g $RG -n $FRAUD_SA -l australiaeast --sku Standard_LRS \
     --enable-hierarchical-namespace true --allow-blob-public-access false
   az storage container create --account-name $FRAUD_SA -n fraud-alerts --auth-mode key
   ```

   > The no-code editor writes to **Azure Data Lake Storage Gen2**, so this is a separate account from the Capture
   > archive.

### Build the job

2. In the portal, open your namespace, then open the `orders-stream` event hub. Under **Features**, select **Process
   data**, and select **Start** on the **Filter and ingest to ADLS Gen2** card. Name the job `asa-fraud-lab7-yourname`
   and select **Create**.

3. Select the **Event Hub** tile and set the input:

   | Setting | Value |
   | --- | --- |
   | Consumer group | Keep the default **Create new** |
   | Serialization | **JSON** |
   | Authentication method | **Managed Identity** (system-assigned) |

   Select **Connect**.

   > **Why a managed identity?** SAS is off, so a connection-string input would fail. When you pick a managed identity,
   > the editor gives the job's identity the **Azure Event Hubs Data Owner** role. This needs your Owner rights. In
   > production, give an input-only job just **Azure Event Hubs Data Receiver**.
   >
   > **Why a new consumer group?** This is an Event Hubs consumer group, and these are not auto-created like Kafka
   > groups. The editor creates one for the job. This keeps its bookmark separate from `amqp_reader.py` on `$Default`.

4. The editor only shows live data when events are flowing. In Cloud Shell, send some events so the fields
   (`eventId`, `customerId`, `sku`, `qty`, `ts`) appear in **Data preview**:

   ```bash
   cd ~/lab7
   export EH_NAMESPACE="$NS.servicebus.windows.net" AUTH=oauth
   for i in 1 2 3; do python producer.py 10 > /dev/null; sleep 10; done
   ```

   If the fields do not appear, select **Autodetect fields**. Check that `qty` has the type **Int**.

### Add the bulk-order rule

5. Select the **Filter** tile. Set **qty** **equals** `3`.

6. Select the **Azure Data Lake Storage Gen2** tile and set the output:

   | Setting | Value |
   | --- | --- |
   | Storage account | `stfraudlab7yourname` |
   | Container | `fraud-alerts` |
   | Authentication method | **Managed Identity** |
   | Path pattern | `bulk-orders/{date}/{time}` |
   | Serialization | **JSON** |

   Select **Connect**. The editor gives the job's identity the **Storage Blob Data Contributor** role on the storage
   account.

### Add the velocity rule

7. In the ribbon, under **Operations**, select **Group by** and connect it to the **Event Hub** tile. Set it up:

   | Setting | Value |
   | --- | --- |
   | Aggregation | **Count** on `eventId` |
   | Group by | `customerId` |
   | Time window | **Tumbling**, `1` minute |

   > A *tumbling window* cuts time into fixed 1-minute blocks. The job counts each customer's orders in each block.

8. Add a second **Filter** tile after **Group by**. Pick the count field that **Group by** creates, and set it
   **greater than** `5`.

9. In the ribbon, add a second **ADLS Gen2** output and connect it to this **Filter** tile. Use the same storage
   account, container and managed identity, but set the **Path pattern** to `velocity/{date}/{time}`.

### Start the job

10. Optional: select **Get static preview** on each tile to check the results. Then select **Save**, then **Start**.
    Set **Output start time** to **Now**, keep the recommended **Streaming units**, and select **Start**.

11. In the event hub, open **Process data** → **Stream Analytics jobs**. Select **Refresh** until the status is
    **Running**. This takes up to 2 minutes.

    > The job only writes events that arrive **after** it starts (because the start time is **Now**). To process old
    > events, choose a **Custom** start time. This is like the `audit-replay` group in Task 4.

### Send events and check the results

12. Send a burst of orders with the Kafka producer:

    ```bash
    cd ~/lab7
    python producer.py 60 > /dev/null
    ```

    > Kafka wrote these events and Stream Analytics reads them over AMQP. The 60 events are spread over 5 customers,
    > so each customer has about 12 orders in the same 1-minute window. If the burst crosses a minute boundary, send
    > another 60.

13. Wait 2–3 minutes (the job writes a file after at most 1 minute), then list the alert files:

    ```bash
    az storage blob list --account-name $FRAUD_SA -c fraud-alerts --auth-mode key \
      --query "[?properties.contentLength > \`0\`].{name:name, bytes:properties.contentLength}" -o table
    ```

    **What you should see:** files under both `bulk-orders/` and `velocity/`.

14. Download one file from each path and look inside:

    ```bash
    for P in bulk-orders velocity; do
      BLOB=$(az storage blob list --account-name $FRAUD_SA -c fraud-alerts --auth-mode key \
        --query "[?starts_with(name, '$P') && properties.contentLength > \`0\`].name | [0]" -o tsv)
      az storage blob download --account-name $FRAUD_SA -c fraud-alerts -n "$BLOB" \
        --auth-mode key --file ~/$P.json --no-progress > /dev/null
      echo "== $P"; head -n 3 ~/$P.json
    done
    ```

    **What you should see:** every `bulk-orders` record has `"qty":3`. Every `velocity` record shows a `customerId`
    with a count greater than `5`.

### Review and stop the job

15. In **Process data** → **Stream Analytics jobs**, select **Open metrics** for your job. **Input events** and
    **Output events** go up after your burst, and **Backlogged input events** stays near zero.

16. In the `orders-stream` event hub, open **Consumer groups**. The group made by the job is listed. The Kafka groups
    from Task 4 are not, because Kafka groups do not show in the portal.

17. **Stop the job** so it stops billing: in **Stream Analytics jobs**, select the job and select **Stop**.

> **Compared with `consumer.py`:** no group-join code, no bookmark code and nothing to deploy. The service scales with
> streaming units. Delivery is still at least once. For ADLS Gen2 output, a preview **Exactly once delivery** option
> is available when you need it.

**Check your work:** alert files exist in both `bulk-orders/` and `velocity/`, and the job is stopped.

---

## Task 7: Choose the right service

**Goal:** know when Event Hubs is the right tool, and when it is not.

Kafka is excellent for high-volume streams, but it is **not** a complete messaging platform. Kafka has no
competing-consumer queue, no rules evaluated on the server for subscribers, no tracking of a message's life cycle,
and no dead-letter queue. Azure's other services cover those: **Service Bus** and **Event Grid**.

1. For each workload below, pick **Event Hubs**, **Service Bus** or **Event Grid**, and write down why. Then compare
   with the answer key.

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
   | 1 | **Event Hubs** | A high-volume partitioned log. Consumer groups give independent readers, and retention allows replay |
   | 2 | **Service Bus queue** | Competing consumers, peek-lock, a maximum delivery count and dead-lettering. Kafka has none of these |
   | 3 | **Service Bus topic + subscriptions** | Each subscription filters on the server. Kafka readers can only choose a position (offset) |
   | 4 | **Event Grid** | It pushes discrete Azure resource events to a handler, with no polling |
   | 5 | **Event Hubs (Kafka endpoint) + Capture** | Existing Kafka clients need only configuration changes, and Capture saves the data to Blob |

   </details>

   > **Going further (not part of this lab):** Kafka compression (gzip), Kafka Streams and Kafka transactions on Event
   > Hubs need the **Premium** or **Dedicated** tier, and Streams and transactions are in preview. Event Hubs also
   > works with Azure Stream Analytics, Azure Databricks, Apache Spark and Apache Flink. If you need Kafka features
   > Event Hubs does not offer, you can run a real Kafka cluster on Azure HDInsight.

---

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| `$RG`, `$NS`, `$HUB` or `$EH_NAMESPACE` is empty | Cloud Shell restarted. Run the "If Cloud Shell restarts" block near the top. |
| `ModuleNotFoundError` | Run the `pip install` command from Task 2, step 2 again. |
| `pip` prints `dependency resolver ... ansible-core ... requires packaging` | Harmless warning about software Cloud Shell already has. Check the install with `python -c "import confluent_kafka, azure.identity, azure.eventhub; print('OK')"`. |
| `$ConnectionString` appears empty in a script | Use the heredoc with `'EOF'` in single quotes, as shown, so the shell does not replace `$ConnectionString` or `$Default`. |
| `SASL authentication error` with SAS in Task 2 | `EH_CONNECTION_STRING` is empty or cut off. Check its length and run Task 2, step 3 again. |
| Producer prints `GETPID ... Failed to acquire idempotence PID ... retrying` | Harmless start-up warning. The producer retries and succeeds. Continue if delivery lines appear and there is no `FAILED:` line. |
| `Authorization failed` or `Topic authorization failed` with Entra ID (Task 3, step 5) | The roles have not taken effect yet (wait 1–3 minutes), or they were assigned at a different scope than the namespace. |
| `DefaultAzureCredential failed ... AudienceNotSupported` | `DefaultAzureCredential` cannot get an Event Hubs token in Cloud Shell. Use `AzureCliCredential` as in `kafka_common.py` and `amqp_reader.py`. |
| `AzureCliCredential.get_token failed: Please run 'az login'` | Run `az login --use-device-code` (Task 3, step 4), even if `az account show` already works, then try again. |
| Consumer exits right away with `processed=0` | The group already saved all its bookmarks. Send more events, or use a different group name. |
| Consumer shows no `ASSIGNED` line | The group is still joining or rebalancing. Wait up to 45 seconds. |
| One consumer shows 0 events in Task 4, Part 2 | The group had not finished rebalancing, or the other consumer had already exited. Run the Part 2 block again. |
| No files in the capture container | Wait for the 60-second capture window to close, then run the list command again. |
| The Process data card or job creation fails | Make sure the event hub is publicly reachable and the `Microsoft.StreamAnalytics` resource provider is registered (**Subscriptions → Resource providers**). |
| Stream Analytics input or output says `Unauthorized` | The job's managed identity needs **Azure Event Hubs Data Owner** (or **Data Receiver**) on the event hub and **Storage Blob Data Contributor** on `stfraudlab7yourname`. Roles can take up to 8 minutes. A connection-string input fails because SAS is off. |
| The Data preview shows no fields | No events are flowing. Run the producer from Task 6, step 4, then select **Autodetect fields**. |
| No files in the `fraud-alerts` container | Check the job is **Running** and that you sent events **after** it started. Files are written after at most 1 minute, so wait 2–3 minutes. Check **Runtime logs** in the editor. |
| Only `bulk-orders` files appear | No customer had more than 5 orders in one 1-minute window, for example because the burst crossed a minute boundary. Run `python producer.py 60` again. |

---

## Cleanup

**Note:** Event Hubs Standard, the storage accounts and the Stream Analytics job (billed per streaming unit while it
runs) cost money while they exist. Stop the job and delete the resources when you finish.

1. Stop any leftover consumers and clear the secrets from your shell:

   ```bash
   pkill -f consumer.py
   unset EH_CONNECTION_STRING AUTH
   ```

2. In the portal, go to **RG-Lab7** (your own resource group).

3. Select **Delete resource group**.

4. Type the resource group name to confirm.

5. Select **Delete**.

---

## Key Takeaways

- **Event Hubs speaks the Kafka protocol, but it is not a Kafka cluster.** Kafka 1.0+ programs connect to one
  address (`<namespace>.servicebus.windows.net:9093`) by changing settings only. There are no brokers or disks to
  manage. Cluster = namespace, topic = event hub. Partitions, consumer groups and offsets work the same way.

- **Use Microsoft Entra ID instead of SAS.** `OAUTHBEARER` with Azure RBAC removes shared secrets, ties every action
  to an identity, and replaces access lists with built-in roles. Turning off local authentication enforces this for
  every client. Owner does not give data access.

- **Partitions give you both order and parallel work.** Order is kept inside a partition, so the key decides which
  events stay in order. The number of partitions is the maximum number of consumers that can work at once in one group.

- **Consumer groups give each reader its own view of the stream.** Each group has its own bookmark, so analytics, fraud
  and audit systems can read and replay the kept events without affecting each other. Kafka groups are created
  automatically. Event Hubs groups (used by AMQP tools) are created by you.

- **Delivery is at least once, so consumers must handle duplicates.** An idempotent producer stops duplicates caused by
  retries, but only a consumer that skips repeats protects you from application resends and crashes before saving a
  bookmark.

- **One stream, many protocols.** Events written with Kafka can be read over AMQP with Azure tools and saved with
  Capture. Teams are not forced to use Kafka tools.

- **No-code stream processing removes consumer code.** A Stream Analytics job reads the stream with a managed identity,
  needs no group-join or bookmark code, and scales with streaming units. Delivery is still at least once.

- **Use the right messaging service for each job.** Event Hubs is for high-volume streams and replay. Service Bus is for
  queues, topics, filtering and dead-lettering. Event Grid is for reacting to discrete events. Most platforms use all
  three.
