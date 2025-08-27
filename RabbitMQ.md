
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


