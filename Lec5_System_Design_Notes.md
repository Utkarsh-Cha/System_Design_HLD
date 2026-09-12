# System Design — Lecture 5: Messaging Queues (Pizza Shop Example)

## Table of Contents
1. [What this lecture is about](#1-what-this-lecture-is-about)
2. [The single pizza shop](#2-the-single-pizza-shop)
   - [How orders are handled](#21-how-orders-are-handled)
   - [The list and the order queue](#22-the-list-and-the-order-queue)
   - [Completion and payment](#23-completion-and-payment)
   - [Why this is asynchronous processing](#24-why-this-is-asynchronous-processing)
   - [Benefit: priority ordering](#25-benefit-priority-ordering)
3. [Scaling up: multiple outlets](#3-scaling-up-multiple-outlets)
   - [What happens when an outlet goes down](#31-what-happens-when-an-outlet-goes-down)
   - [Why an in-memory list is not enough](#32-why-an-in-memory-list-is-not-enough)
4. [Multi-server architecture with a database](#4-multi-server-architecture-with-a-database)
   - [The setup](#41-the-setup)
   - [A server crashes — how do we reroute its orders?](#42-a-server-crashes--how-do-we-reroute-its-orders)
   - [The notifier and the heartbeat](#43-the-notifier-and-the-heartbeat)
5. [The duplication problem](#5-the-duplication-problem)
   - [Solution: load balancing](#51-solution-load-balancing)
   - [Why consistent hashing prevents duplicates](#52-why-consistent-hashing-prevents-duplicates)
6. [Putting it all together: the message / task queue](#6-putting-it-all-together-the-message--task-queue)
7. [Examples of messaging queues](#7-examples-of-messaging-queues)
8. [Final takeaway](#8-final-takeaway)

---

## 1. What this lecture is about

This lecture introduces **messaging queues** as a system design concept, using a standard **pizza shop** as the running example. The pizza shop is built up step by step — from a single shop, to a chain of outlets, to a multi-server architecture with a database — and at each step a new problem appears whose solution is eventually found to already be bundled inside a message queue.

---

## 2. The single pizza shop

### 2.1 How orders are handled

In a normal pizza shop, someone is always taking orders. Crucially, **the shop does not stop taking new orders while pizzas are being made**. Multiple clients keep requesting pizzas, and each of them gets a response **immediately** — but that response is not the pizza. The response is something like *"please sit down"* or *"can you come back after some time"*.

This is the key idea:

> You relieve the client from expecting an immediate result by giving them a response which is **not the pizza**, but a **confirmation that the order has been placed**.

The interaction for a client looks like this:

```
CLIENT-1  ---- REQUEST ---->  PIZZA SHOP
CLIENT-1  <-- RESPONSE: OK --  PIZZA SHOP     (acknowledgement, not the pizza)

CLIENT-2  ---- REQUEST ---->  PIZZA SHOP
CLIENT-2  <--- RESPONSE ----  PIZZA SHOP
```

### 2.2 The list and the order queue

Because the pizza is not delivered at the moment of the request, the shop needs somewhere to remember what has been asked for. What you need is a **list**, which maintains order number 1, order number 2, order number 3, and so on.

Once you are maintaining this list, the flow becomes: note down the order → start making pizzas. When the second client comes in, you can either stop making the current pizza, or — if you have multiple people — one person takes the order and simply **adds it to the queue of pizzas to be made**.

So there is a queue of pizza orders `PO1`, `PO2`, … and it is mirrored by the list. (The lecture notes that these may in fact be the *same* queue.)

```
            QUEUE OF PIZZA ORDERS
          [ PO1 | PO2 |  ...  ]
                    |
   LIST             v
  +---+        +-------------+
  | 1 |  <-->  | PIZZA SHOP  |
  | 2 |        +-------------+
  | 3 |          ^        |
  | 4 |   REQUEST|        | RESPONSE: OK
  +---+          |        v
              CLIENT-1 / CLIENT-2
```

While you are working on one pizza, you can take **as many orders as you like**. It is a very simple architecture.

### 2.3 Completion and payment

When a pizza is done, you **remove it from the queue** (which also removes it from the list). Then you ask the client to pay, and the client sends back the money.

```
PIZZA SHOP  ---- PAY ---->  CLIENT-1
PIZZA SHOP  <-- MONEY ----  CLIENT-1
```

At that point the client is entirely relieved.

### 2.4 Why this is asynchronous processing

The special thing about this whole arrangement is that it is **asynchronous**: you did **not** make the client wait for your response — neither for the pizza nor for the payment step.

What this buys you:

- The client can **do other tasks** during that time — checking their phone, going outside, whatever. It does not matter what they do.
- The client is happier, because they can **distribute their resources elsewhere** instead of focusing only on you.

### 2.5 Benefit: priority ordering

Asynchronous processing also helps **you**, the pizza maker: it lets you **order your tasks according to their priority**.

- There might be one pizza that has to be made immediately.
- There might be something very easy to do, like filling up a Coke can.

Those orders can be placed in the queue according to their priority. So you are able to **manipulate the queue by priority**, and at the same time allow clients to spend their time more judiciously — all just by using asynchronous processing.

---

## 3. Scaling up: multiple outlets

### 3.1 What happens when an outlet goes down

Now consider a very good scenario: you become super successful and you have a **chain** of outlets — something like Domino's. Say you have Pizza Shop 1, Pizza Shop 2 and Pizza Shop 3, each with multiple clients connected to it.

```
   PS1        PS2        PS3 (X — down)
    |          |          |
 clients    clients    clients  ---> redirected to PS1 and PS2
```

Now assume the worst: **one of the shops actually goes down** — a power outage, or anything else. Say Pizza Shop 3 goes down.

- The **takeaway orders** of that shop are easy to handle: you just dump them.
- The **delivery orders**, however, can actually be **sent to the other shops**. Those shops can complete them, and you still save some money.

So the clients that were connected to PS3, along with their orders, must now be connected to the remaining shops, and their orders must be transferred there.

### 3.2 Why an in-memory list is not enough

How do you actually do this? The simple approach of **maintaining a list in memory does not work**, because once the shop is down it loses electricity and the computer shuts down — the list disappears with it.

You therefore need **persistence** in your data, which means you need a **database**, and the list of orders has to be stored in that database.

---

## 4. Multi-server architecture with a database

### 4.1 The setup

The new (and slightly more complicated) architecture has:

- A set of servers: **S0, S1, S2, S3** (four servers), each with its own queue of orders.
- A **database** storing the list of all orders, with columns: **order ID, contents, and whether it is done or not**.

```
   SERVERS                                  DB
 +---------+                    +----+-----------+-------+
 | S0 : 20 |                    | ID | Contents  | Done? |
 +---------+                    +----+-----------+-------+
 | S1 : 8  |                    | 1  | Pepperoni |   Y   |
 +---------+                    | 2  | Ham       |   N   |
 | S2 : 3  |                    | 3  | Cheese    |   N   |
 +---------+                    | 4  | ...       |   N   |
 | S3 : 9, 11 |                 +----+-----------+-------+
 +---------+
```

The servers receive orders as follows: order 20 → S0, order 8 → S1, order 3 → S2, and orders 9 and 11 → S3. The full set of live order numbers is `3, 8, 20, 9, 11, …`.

### 4.2 A server crashes — how do we reroute its orders?

Now **S3 crashes**. Orders **9 and 11** need to be rerouted somewhere. How?

**One way:** check in the database which orders belong to S3. For this to work, every time an entry is made you must also **note down the server ID that is handling that order**.

But this is getting complicated. So instead of tracking server IDs per order, the lecture introduces a different component.

### 4.3 The notifier and the heartbeat

Introduce a **notifier**, sitting between the servers and the database, which checks for a **heartbeat** in each server.

```
  S0  <---\
  S1  <----\
  S2  <-----( NOTIFIER )-------> DB
  S3  <----/   Load Balancing
    (X)        Heart-Beat
```

How it works:

- The notifier **talks to each server and asks whether it is alive**, every 10 seconds or every 15 seconds.
- If a server **does not respond**, the notifier assumes that server is **dead**.
- If it is dead, it cannot handle orders. The notifier then **queries the database to find all orders which are not done**.
- It picks those orders and **distributes them to the remaining 3 servers**.

---

## 5. The duplication problem

There is a problem with the scheme above: **what if there is duplication?**

Consider order number 3. It is not yet done, so it is picked up by the query on the database — the query returns `3, 8, 20, 9, 11` and all of them get distributed.

Suppose order number 3 this time goes to **server 1**. But **server 2 already had order number 3 and is processing it**. Server 2 is going to end up making that pizza and sending it to the same place that S1 is sending its pizza to. The result: **a big loss and lots of confusion**.

### 5.1 Solution: load balancing

The fix is to use some sort of **load balancing**.

Load balancing *seems* like it is only about sending the right amount of load to each server, but the principles of load balancing also ensure that you **do not have duplicate requests**. **Consistent hashing** is one technique by which you can get rid of duplicates as well.

So that single principle takes care of two things:

1. Balancing the load.
2. Not sending duplicates.

*(The lecture refers to a separate video on load balancing for the details of consistent hashing, and recommends watching it.)*

### 5.2 Why consistent hashing prevents duplicates

The reason is that each server handles a **set of buckets**: S1 handles a set of buckets, S2 handles a set of buckets, and so on.

- When a server crashes, **S2 does not lose its buckets** — it only gets **new buckets added** to it. The same holds for S1.
- Therefore order number 3 will **never** be reassigned, because it belongs to S2 even now.
- Meanwhile the orders from the crashed server, 9 and 11, may come into S1 and S2 as newly added buckets.

So, through **load balancing** plus some sort of **heartbeat mechanism**, you can notify all the failed orders to the new servers.

---

## 6. Putting it all together: the message / task queue

At this point the architecture needs four features at once:

1. **Assignment / notification**
2. **Load balancing**
3. **Heartbeat**
4. **Persistence**

What if you want all of these in one thing? **That would be a message queue.**

For this use case it is not so much a *message* queue as a **task queue**. What it does:

- Takes tasks.
- **Persists** them.
- **Assigns** them to the correct server.
- **Waits for them to complete.**
- If the server is taking too long to give an **acknowledgement**, it concludes that the server is dead and **assigns the task to the next server**.

There are, of course, **multiple strategies for assigning** tasks — just as load balancing has multiple strategies — but all of that is **encapsulated by the task queue**.

This is an important system design concept: using **messaging queues / task queues** to get work done easily, so that all that complexity is encapsulated into just one thing. The pizza example is a little extreme because it needs *everything* together — which is exactly why a pizza shop would want a task queue.

---

## 7. Examples of messaging queues

- **RabbitMQ**
- **ZeroMQ** — a library which lets you write a messaging queue quite easily
- **JMS** — Java Messaging Service
- Amazon also has a few messaging queues

They are really good encapsulations for complexities on the server side.

---

## 8. Final takeaway

This is a **fundamental concept of system design** rather than a specific system that has been designed. But if you are going to build a pizza shop, a message/task queue seems like a good idea.

**Concept chain of the lecture:**

```
Asynchronous processing (immediate ack, not the result)
        -> need a list/queue of pending work
        -> multiple outlets/servers, one crashes
        -> need persistence  ..........  DATABASE
        -> need failure detection  .....  HEARTBEAT + NOTIFIER
        -> need duplicate-free reassignment  ....  LOAD BALANCING
                                                 (consistent hashing)
        -> all four in one component  ...  MESSAGE / TASK QUEUE
```
