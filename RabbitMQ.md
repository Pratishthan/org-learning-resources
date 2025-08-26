
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

## 7. Hands-on Exercise for Participants

* Change the producer to send multiple messages in a loop (e.g., 5 messages).
* Observe how the consumer processes them one by one.
* Open the RabbitMQ UI and watch the queue length increase and decrease.
* Stop the consumer and send messages again — notice how the queue retains the messages until the consumer is restarted.

---



