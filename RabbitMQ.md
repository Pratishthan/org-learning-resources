
# 📘 Session 1: Introduction to RabbitMQ and Building the First Queue

## 1. What is RabbitMQ?

RabbitMQ is a **message broker**, meaning it acts as a middleman that takes messages produced by one application (called a producer) and delivers them to one or more applications that want to receive them (called consumers). Instead of services directly calling each other, they can communicate indirectly through RabbitMQ. This helps decouple systems, improves reliability, and allows applications to handle spikes in traffic more gracefully.

RabbitMQ implements AMQP (Advanced Message Queuing Protocol). AMQP is a protocol that helps in communication between services using messages.

Think of RabbitMQ as a **post office**:

* The producer is the person sending a letter.
* The consumer is the person receiving the letter.
* RabbitMQ is the post office that holds and forwards the mail.
* The queue is the mailbox where letters wait until they are picked up.

---

## 2. Key Concepts

Before coding, participants should understand these terms:

* **Broker**: The RabbitMQ server itself.
* **Queue**: A storage buffer inside RabbitMQ where messages wait until a consumer picks them up.
* **Producer**: An application that sends messages to a queue.
* **Consumer**: An application that reads messages from a queue.
* **Message**: The actual data sent (text, JSON, binary, etc.).
* **Connection**: A TCP connection between the application and RabbitMQ.
* **Channel**: A lightweight communication path inside a connection. Multiple channels can share a single connection.

Why channels? 

* A single TCP connection can host **many channels** (hundreds or thousands).
* Each channel is independent and can act like its own mini-connection for queues, exchanges, publishing, and consuming.
* Channels are **much lighter** than connections.

👉 **Analogy:** If the *connection* is the highway, then *channels* are the lanes on that highway. Instead of building 100 highways, you can build 1 highway with 100 lanes.

---

## 3. Setting up RabbitMQ

For this session, we use Docker to quickly start a RabbitMQ instance:

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  rabbitmq:3-management
```

* Port **5672** is for applications to connect.
* Port **15672** is for the Management UI.

You can access the UI at [http://localhost:15672](http://localhost:15672) with credentials **guest / guest**.
In the UI, you will see sections for **Queues**, **Exchanges**, **Connections**, and **Channels**.

---

## 4. First Queue Example: Producer

We will now write a producer in Node.js using the **amqplib** library.

Install dependency:

```bash
npm install amqplib
```

Producer code (`producer.js`):

```js
const amqp = require('amqplib');

async function main() {
  // 1. Connect to RabbitMQ
  const connection = await amqp.connect('amqp://localhost');
  
  // 2. Create a channel
  const channel = await connection.createChannel();
  
  // 3. Declare a queue named "hello"
  const queue = 'hello';
  await channel.assertQueue(queue, { durable: false });
  
  // 4. Send a message
  const message = 'Hello RabbitMQ!';
  channel.sendToQueue(queue, Buffer.from(message));
  
  console.log(`Sent: ${message}`);
  
  // 5. Close the channel and connection
  await channel.close();
  await connection.close();
}

main().catch(console.error);
```

Explanation:

* We first establish a **connection** to the RabbitMQ broker.
* Then, we create a **channel** to communicate with the broker.
* We call `assertQueue("hello")` to make sure the queue exists (this creates it if not).
* We send a message with `sendToQueue()`.
* The message is stored in RabbitMQ until a consumer picks it up.

---

## 5. First Queue Example: Consumer

Consumer code (`consumer.js`):

```js
const amqp = require('amqplib');

async function main() {
  // 1. Connect to RabbitMQ
  const connection = await amqp.connect('amqp://localhost');
  
  // 2. Create a channel
  const channel = await connection.createChannel();
  
  // 3. Declare the same queue "hello"
  const queue = 'hello';
  await channel.assertQueue(queue, { durable: false });
  
  // 4. Consume messages from the queue
  console.log(`Waiting for messages in queue: ${queue}`);
  channel.consume(queue, msg => {
    console.log(`Received: ${msg.content.toString()}`);
    channel.ack(msg); // acknowledge message so it is removed from the queue
  });
}

main().catch(console.error);
```

Explanation:

* Like the producer, the consumer creates a connection and channel.
* It ensures the same queue exists with `assertQueue("hello")`.
* It subscribes to the queue with `consume()`.
* Every time a new message arrives, RabbitMQ delivers it to this consumer, and we print it.
* We use `ack()` to tell RabbitMQ the message has been processed successfully.

---

## 6. Demonstration Flow

1. Run the consumer first:

   ```bash
   node consumer.js
   ```

   The program waits for messages.

2. Run the producer:

   ```bash
   node producer.js
   ```

   This sends a message and then exits.

3. The consumer will display:

   ```
   Received: Hello RabbitMQ!
   ```

4. In the Management UI, you can see the queue "hello" and observe the message count before and after consumption.

---
 # Session 2 — Work Queues (Task distribution) & Fair Dispatch — explained in great detail



## 1) Core concepts (detailed)

**Work queue**: A queue where producers push tasks (jobs) and multiple worker processes consume tasks and perform work. The broker holds tasks until a worker acknowledges successful processing.

**Durable queue** (`durable: true`): The queue definition survives broker restart. Alone it’s not enough to persist messages — messages must be published as persistent too.

**Persistent messages** (`{ persistent: true }` / `delivery_mode=2`): Mark messages so the broker will try to write them to disk. Combine with durable queue for persistence across restarts.

**Manual acknowledgements** (`noAck:false`, `ch.ack(msg)`): Consumer tells the broker when a message has been successfully processed. Until acked, the message is “unacked” and will not be removed. If the consumer dies, the broker requeues the unacked message and delivers it to another consumer.

**Prefetch (QoS)** (`ch.prefetch(n)`): Limits how many unacknowledged messages a consumer can have at a time. `prefetch(1)` is the classic *fair dispatch* setting — the broker will not send a new message to the consumer until it acks the previous one. This prevents one fast producer from overwhelming a single slow worker.

**Redelivered flag**: When a message is re-delivered (because it was unacked and the consumer died), `msg.fields.redelivered` is true. Useful to detect retries/poison messages.

**At-least-once delivery**: With acks and re-deliveries, RabbitMQ guarantees at-least-once delivery — meaning workers must be idempotent because the same message could be delivered multiple times.

---

## 3) Example code — robust production-style producer and worker

### Producer (durable queue + persistent messages + confirm channel)

```js
// producer.js
const amqp = require('amqplib');

async function sendTasks(tasks = []) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createConfirmChannel();

  const queue = 'task_queue';
  // Durable queue: survives broker restart
  await ch.assertQueue(queue, { durable: true });

  for (const task of tasks) {
    const content = Buffer.from(JSON.stringify(task));
    // Mark message persistent
    ch.sendToQueue(queue, content, { persistent: true }, (err, ok) => {
      if (err) console.error('Publish failed for task', task, err);
      else console.log('Published task', task.id || '(no id)');
    });
  }

  // Wait for all confirms before closing
  await ch.waitForConfirms();
  await ch.close();
  await conn.close();
}

sendTasks([
  { id: 1, type: 'resize', image: 'a.jpg' },
  { id: 2, type: 'resize', image: 'b.jpg' },
]).catch(console.error);
```

Key points:

* `createConfirmChannel()` + `waitForConfirms()` ensures the broker acknowledged persistence. This reduces risk of losing messages due to publisher crash.
* Queue is declared durable; messages are persistent.

---

### Worker (prefetch, manual ack, safe error handling)

```js
// worker.js
const amqp = require('amqplib');

async function startWorker(concurrency = 1) {
  const conn = await amqp.connect('amqp://localhost');
  const ch = await conn.createChannel();

  const queue = 'task_queue';
  await ch.assertQueue(queue, { durable: true });

  // Fair dispatch: limit unacked messages per worker
  ch.prefetch(concurrency);

  console.log('Worker started, concurrency:', concurrency);

  ch.consume(queue, async (msg) => {
    if (msg === null) return;
    const content = JSON.parse(msg.content.toString());
    const redelivered = msg.fields.redelivered;

    try {
      console.log('Received task', content.id, 'redelivered?', redelivered);
      // >>> DO WORK HERE (simulate with async function)
      await doWork(content); // your async job handler

      // On success:
      ch.ack(msg);
      console.log('Acked', content.id);
    } catch (err) {
      console.error('Processing error for', content.id, err);

      // Decide whether to requeue or send to DLQ.
      // Example: if transient error, requeue; if permanent, nack and drop/reroute.
      const shouldRequeue = decideRequeue(err, msg);

      // nack with requeue true/false
      ch.nack(msg, false, shouldRequeue);
      console.log('Nacked', content.id, 'requeue=', shouldRequeue);
    }
  }, { noAck: false });
}

async function doWork(task) {
  // Example: simulate I/O-bound work
  await new Promise(res => setTimeout(res, 1000));
  if (task.fail) throw new Error('simulated failure');
  return true;
}

function decideRequeue(err, msg) {
  // Simple heuristic: if redelivered already (retry happened), don't requeue
  // to avoid infinite retry loops; send to DLQ instead (handled by routing).
  if (msg.fields.redelivered) return false;
  // otherwise requeue once
  return true;
}

startWorker(1).catch(console.error);
```

Key points:

* `ch.prefetch(concurrency)` limits unacked messages. If `concurrency` is 1 (classic fair dispatch), the broker won't deliver another message to this worker until it acked the current one.
* Worker acknowledges only after successful processing.
* On error, worker can `nack` and decide to requeue (true) or not (false). If not requeued, the message is re-routed to dead-letter exchange (if configured) or dropped.
* Use `msg.fields.redelivered` to detect retries.

---

Perfect 👍 Let’s now deep-dive into **Session 3: Exchanges in RabbitMQ**.
This is a very important session because **exchanges are the “routing brains”** of RabbitMQ — they decide **how messages published by producers end up in one or more queues**.

---

# 📘 **Session 3: Exchanges in RabbitMQ**

---

## 1. **Why Exchanges?**

* In Session 1 & 2 we saw **queues**: a simple container where messages are sent and consumed.
* But if producers could only publish directly to queues, that would be limiting:

  * What if the same message needs to go to **multiple consumers**?
  * What if we need different types of consumers to get different subsets of messages?
  * What if we want **flexible routing** rules without modifying producers?

That’s where **Exchanges** come in.

👉 In RabbitMQ, producers **never publish directly to queues**. They publish to an **exchange**, and the exchange decides which queue(s) the message should go to (based on binding rules).

---

## 2. **How it Works (Flow)**

1. **Producer** → sends a message to an **Exchange**.
2. **Exchange** → applies its routing logic using **bindings** and possibly **routing keys**.
3. **Queue(s)** → receive the message if the routing rules match.
4. **Consumers** → consume messages from those queues.

Think of an **Exchange** like a *post office sorting center*:

* You don’t deliver letters straight into people’s houses (queues).
* You deliver them to the post office (exchange).
* The post office sorts them based on addresses/rules and delivers them to the right mailboxes (queues).

---

## 3. **Types of Exchanges**

RabbitMQ provides **four built-in exchange types** (each with different routing strategies):

### (a) **Direct Exchange**

* Routes messages to queues **based on exact matching of a routing key**.
* Each queue is **bound** to the exchange with a binding key.
* When a message is published with a routing key, the exchange delivers it to the queue(s) with a matching binding key.
```js
const exchange = 'direct_logs';
await channel.assertExchange(exchange, 'direct', { durable: false });

channel.publish(exchange, 'error', Buffer.from('Error log'));

```
📌 Example:

* Queue1 bound with key = `error`
* Queue2 bound with key = `info`
* Producer publishes message with routing key `error` → goes to Queue1.

👉 Useful for **point-to-point routing** or selective consumption.

---

### (b) **Fanout Exchange**

* Ignores routing keys completely.
* Sends the message to **all queues bound to it** (broadcast).
* Think of it as a **loudspeaker**.

📌 Example:

* Three queues bound to a fanout exchange.
* Producer sends one message → all three queues receive a copy.

👉 Useful for **pub/sub systems** like notifications, live updates, event broadcasting.
```js
const exchange = 'logs';
await channel.assertExchange(exchange, 'fanout', { durable: false });

channel.publish(exchange, '', Buffer.from('Broadcast message'));
```
---

### (c) **Topic Exchange**

* Routes messages to queues based on **pattern matching** with routing keys.
* The routing key is a string with words separated by dots (e.g., `logs.error.database`).
* Bindings can use wildcards:

  * `*` (matches exactly one word)
  * `#` (matches zero or more words)

📌 Example:

* Queue1 bound with `logs.error.*`
* Queue2 bound with `logs.#`
* Message with key `logs.error.database` → goes to both queues.

👉 Useful for **complex routing** like event classification, logging, or multi-service messaging.
```js
const exchange = 'topic_logs';
await channel.assertExchange(exchange, 'topic', { durable: false });

channel.publish(exchange, 'order.created', Buffer.from('New order event'));
```
---

### (d) **Headers Exchange**

* Routing happens based on **message headers**, not routing keys.
* You define a binding with a set of key-value pairs.
* Messages with matching headers are delivered to that queue.

📌 Example:

* Queue1 bound with `{ format: "pdf", type: "report" }`
* If a message has these headers, it goes to Queue1.

👉 Useful when **metadata in headers** decides routing rather than a string routing key. Less common but powerful.
```js
const exchange = 'headers_logs';
await channel.assertExchange(exchange, 'headers', { durable: false });

channel.publish(exchange, '', Buffer.from('Message with headers'), {
  headers: { severity: 'high', format: 'json' }
});
```
---

## 4. **Default Exchange**

* RabbitMQ has a built-in **default direct exchange** (`""` — empty string as the name).
* Any message sent to this exchange with a routing key equal to a queue’s name is delivered directly to that queue.
* That’s why in Session 1 we could do `channel.sendToQueue('queue', msg)` without declaring an exchange — it was using the default one.

---

## 5. **Bindings and Routing Keys**

### What are Bindings?

* A **binding** connects a queue to an exchange.
* It defines the **rule** (routing key or pattern) that decides if a message from that exchange should be delivered to that queue.

### Routing Key

* A **routing key** is a string set by the producer when publishing.
* Exchanges (depending on type) use the routing key to determine where the message should go.

**Example:**

```js
await channel.assertExchange('direct_logs', 'direct', { durable: false });
const q = await channel.assertQueue('', { exclusive: true });

// bind queue to exchange with specific keys
await channel.bindQueue(q.queue, 'direct_logs', 'info');
await channel.bindQueue(q.queue, 'direct_logs', 'error');

// producer side
channel.publish('direct_logs', 'info', Buffer.from('This is an info log'));
```

Here:

* Only messages with routing key `info` or `error` reach this queue.
* If producer publishes with `debug`, this queue won’t receive it.

---

## 6. **Publish/Subscribe Pattern**

### Concept

* A common pattern where **one producer sends a message that multiple consumers receive**.
* Implemented with a **fanout exchange**.
* All bound queues get a copy of the message.

### Use Case

* Event-driven systems: logging, monitoring, notifications, real-time updates.

**Example (Node.js):**

```js
const exchange = 'logs';
await channel.assertExchange(exchange, 'fanout', { durable: false });

// declare a queue for this consumer
const q = await channel.assertQueue('', { exclusive: true });

// bind queue to the fanout exchange
await channel.bindQueue(q.queue, exchange, '');

// consumer
channel.consume(q.queue, (msg) => {
  console.log(" [x] Received:", msg.content.toString());
}, { noAck: true });

// producer
channel.publish(exchange, '', Buffer.from('System started!'));
```

---

Great 👍 Let’s now dive deep into **Session 4: Reliability in RabbitMQ**.
This session builds on the basics (Session 2) and advanced routing (Session 3) by focusing on **making RabbitMQ reliable** for production systems.

---

# **Session 4: Reliability in RabbitMQ **

Reliability means **no message is lost**, and **processing happens exactly as intended**, even if there are failures.

In this session, we cover:

1. Message Acknowledgments (`ack`, `nack`, `reject`)
2. Dead Letter Queues (DLQ)
3. Message TTL (time-to-live)
4. Publisher Confirms
5. Handling Consumer Failures

---

## **1. Message Acknowledgments**

### Why Acknowledgments?

* By default, RabbitMQ delivers messages to consumers **and assumes success immediately**.
* If a consumer crashes while processing, messages may be lost.
* Acknowledgments (`ack`) tell RabbitMQ:

  * ✅ message was processed successfully → remove from queue
  * ❌ message failed → requeue or discard

### Types of Acknowledgments:

1. **Manual Acknowledgment (`ack`)**

   * Explicitly confirm message was processed.
   * Example: After writing to DB, send `ack`.
   * Node.js:

     ```js
     channel.consume('task_queue', (msg) => {
       console.log(" [x] Received:", msg.content.toString());
       setTimeout(() => {
         console.log(" [x] Done");
         channel.ack(msg);  // acknowledge only after processing
       }, 2000);
     }, { noAck: false });
     ```

2. **Negative Acknowledgment (`nack`)**

   * Tells RabbitMQ that processing **failed**.
   * Options:

     * `requeue: true` → put message back in the same queue
     * `requeue: false` → discard message or send to **DLQ**
   * Example:

     ```js
     try {
       processMessage(msg);
       channel.ack(msg);
     } catch (error) {
       channel.nack(msg, false, true); // requeue the message
     }
     ```

3. **Reject**

   * Like `nack` but only for a **single message** (not multiple).
   * Example: `channel.reject(msg, false);`

---

## **2. Dead Letter Queues (DLQ)**

### Concept

* A **DLQ** is a special queue for messages that **cannot be processed**.
* Reasons messages go to DLQ:

  1. Consumer rejects (`nack` with `requeue=false`)
  2. Message TTL expired
  3. Queue length limit exceeded

### Why DLQ is important?

* Prevents “poison messages” (bad data that keeps failing) from blocking processing.
* Allows debugging and monitoring of failed messages.

### Node.js Example:

```js
// Dead-letter exchange
await channel.assertExchange('dlx', 'direct', { durable: true });
await channel.assertQueue('dead_letter_queue', { durable: true });
await channel.bindQueue('dead_letter_queue', 'dlx', 'failed');

// Normal queue with DLQ settings
await channel.assertQueue('task_queue', {
  durable: true,
  deadLetterExchange: 'dlx',
  deadLetterRoutingKey: 'failed'
});

// If consumer rejects, message goes to DLQ
```

---

## **3. Message TTL (Time-To-Live)**

### Concept

* A message can have a **time-to-live (TTL)**, after which it is considered “expired.”
* Expired messages → removed or sent to DLQ (if configured).

### Why TTL?

* Useful for:

  * Temporary tasks (like session cleanup).
  * Expiring cache-like messages.
  * Avoiding queues growing infinitely with outdated data.

### Two Types of TTL:

1. **Queue-level TTL** → applies to all messages in the queue.

   ```js
   await channel.assertQueue('temp_queue', { messageTtl: 10000 }); // 10s
   ```

2. **Per-message TTL** → set when publishing.

   ```js
   channel.sendToQueue('temp_queue', Buffer.from('Expires soon'), {
     expiration: 5000 // 5s
   });
   ```

---

## **4. Publisher Confirms**

### The Problem

* When producers publish messages, how do they **know the message was actually stored in RabbitMQ**?
* If the broker crashes before saving, messages can vanish.

### Solution → Publisher Confirms

* With **confirm channels**, producers get an **ack/nack** back from RabbitMQ.
* Ensures **at least once delivery**:

  * `ack` → broker persisted message
  * `nack` → broker failed, try again

### Node.js Example:

```js
const confirmChannel = await connection.createConfirmChannel();

confirmChannel.sendToQueue('task_queue', Buffer.from('Important message'), {}, (err, ok) => {
  if (err) {
    console.error("Message NOT confirmed!", err);
  } else {
    console.log("Message confirmed by RabbitMQ");
  }
});
```

This way, the producer knows whether to retry or proceed.

---

## **5. Handling Consumer Failures**

### Why?

* Consumers can crash, disconnect, or fail to process messages correctly.
* RabbitMQ provides mechanisms to handle these failures gracefully.

### Strategies:

1. **Manual Acknowledgments**

   * Don’t `ack` until message is fully processed.
   * If consumer dies, RabbitMQ re-delivers the message to another consumer.

2. **Retry Mechanism**

   * Consumers can `nack` and requeue a message a few times before sending it to DLQ.
   * Example:

     ```js
     if (msg.fields.deliveryTag < 3) {
       channel.nack(msg, false, true); // retry
     } else {
       channel.nack(msg, false, false); // move to DLQ
     }
     ```

3. **Dead Letter Queues**

   * Store failed messages for later inspection, instead of retrying infinitely.

4. **Consumer Prefetch**

   * Use `channel.prefetch(1)` to limit the number of unacked messages per consumer.
   * Prevents one slow worker from being overloaded.
   * Example:

     ```js
     channel.prefetch(1);
     ```

---
