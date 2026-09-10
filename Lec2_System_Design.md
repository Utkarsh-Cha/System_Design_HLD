# Lecture 2: System Design Basics

> **Who this lecture is for:** Anyone who has never designed a system before. The instructor calls this "probably the place to start."
>
> **Note on these notes:** Everything here comes from the lecture (spoken explanation + whiteboard). Where the lecture uses a technical term without explaining it, a short explanation is added and marked **(Clarification)** so you can tell it apart from what the instructor said.

---

## Contents

1. The starting point: your code on your computer
2. Sharing your code with others: API, request, response
3. Problems with running the service on your own desktop
4. Moving to the cloud
5. More users arrive: the need for scalability
6. The two ways to scale: vertical and horizontal
7. Vertical vs horizontal: the 5 key differences
8. What is used in the real world: the hybrid approach
9. The 3 major considerations of system design, and trade-offs
10. Key terms (quick glossary)
11. Revision and interview questions

---

## 1. The Starting Point: Your Code on Your Computer

### The situation

Imagine you have a computer, and on it you have written an **algorithm**. So some **code** is running on this computer.

On the whiteboard, the instructor draws a desktop monitor with `CODE` written on the screen, and a thought bubble above it containing a **graph (nodes connected by edges)**. The graph is just a visual way of saying "some algorithm lives here." It is not about graphs specifically.

### What the code does

This code behaves like a **normal function**:

```text
   input  ──►  [ your code / algorithm ]  ──►  output
```

It takes some input and gives out an output. Nothing more complicated than that.

### Why this matters

People look at your code and decide it is **really useful to them**. They are even **ready to pay you** so that they can use it.

This is the seed of every real system: *you have something useful, and many people want to use it.* The rest of the lecture is about how to make that possible.

---

## 2. Sharing Your Code with Others: API, Request, Response

### The problem

You **cannot go around giving your computer to everybody**. People need a way to use your code without physically having your machine.

### The solution: expose the code over the internet

You **expose** your code using some **protocol** that runs on the **internet**. The way you expose it is through something called an **API**.

- **API = Application Programming Interface**
- It is the "doorway" through which other people's devices can send input to your code and get the output back.

### What changes about the output

Normally, when code runs, you might store its output in a file or in a database. Here, instead of storing the output, you **return it** to the person who asked for it.

### Request and Response

| Term | What it is | Direction |
|---|---|---|
| **Request** | The thing that is sent **to you**. It is what people *request* from you (their input). | Client ──► Server |
| **Response** | The output your code produces and **sends back**. | Server ──► Client |

**Key rule:** For **each request**, there is a **corresponding response** that your computer sends back.

### Whiteboard diagram

```text
                        (thought bubble: algorithm as a graph)
                                     o──o
                                      \ │
                                       o

  ┌────────┐      REQUEST  ───────►   ┌──────────────┐
  │ Mobile │                          │   SERVER     │
  │ phone  │   ◄───────  RESPONSE     │   (CODE)     │
  │(client)│                          └──────────────┘
  └────────┘
        ▲  ▲
        │  │
       API  (points to both the REQUEST and RESPONSE arrows)

                    EXPOSE
                   INTERNET
```

What the diagram shows:

- A **client** (drawn as a mobile phone) is the device of the person using your service.
- The computer running your code is now called the **SERVER** (because it *serves* requests).
- `API` is written with arrows pointing at both `REQUEST` and `RESPONSE`, meaning **the API is what defines and carries this request–response exchange**.
- `EXPOSE` and `INTERNET` are written below the server, meaning **the server's code is exposed to the world over the internet**.

---

## 3. Problems with Running the Service on Your Own Desktop

Imagine actually setting up this computer yourself to serve paying customers. You would have to handle a lot of things:

1. **Database:** It might need a database connected to it, set up within the desktop itself. (On the whiteboard, a database cylinder is drawn connected to the right side of the server.)
2. **Endpoints:** You would need to configure the **endpoints** that people connect to.
   - **(Clarification)** An *endpoint* is a specific address on your server that a client sends a request to, e.g. one URL for one feature of your API.
3. **Power loss / failures:** What if there is a power cut? What if someone **pulls the plug**?

### Why this is serious

> You **cannot afford to have your service go down**, because lots of people are **paying money** for it.

A desktop sitting in your room is fragile. One unplugged cable and every paying customer loses access. This is the reasoning that leads to the cloud.

---

## 4. Moving to the Cloud

### The instructor's recommendation

> You should host your services on the **cloud**.

### What is the difference between a desktop and the cloud?

The instructor's answer: **"Nothing really."**

- The **cloud is a set of computers that somebody provides to you, for money.**
- Example: **AWS (Amazon Web Services)**, the most popular cloud provider.
- If you pay them, they give you **computation power**.
- **Computation power** is nothing but **a desktop that they have somewhere** which can run your algorithm.
- Technically they are not necessarily "desktops," just **computers** that you can use to run your service.

### How do you put your code on that computer?

You can do something like a **remote login** into that computer, and then store/run your algorithm there, just like you would on your own machine.

### Whiteboard change

The instructor **erases the stand of the desktop monitor** and **draws a cloud around the SERVER box**. This visually says: *it is the same server, it just lives inside the cloud now instead of on your desk.* `AWS` is written at the bottom right and then erased, because AWS was only an example of a provider.

```text
  ┌────────┐     REQUEST  ──►     .-~~~~~~~~~~~~-.
  │ Client │                     (  ┌──────────┐  )
  │        │  ◄──  RESPONSE      (  │  SERVER  │  )
  └────────┘                      (  └──────────┘ )
                                   '-~~~~~~~~~~~-'
                                      CLOUD (e.g. AWS)
```

### WHY we prefer the cloud

> The **configuration**, the **settings**, and the **reliability** can be taken care of **to a large extent by the solution providers.**

So the problems from Section 3 (setup, configuration, power loss) are mostly handled by the cloud company.

### What this frees you to do

Now that the server is hosted on the cloud (on "some computer that we don't know about"), **you can focus on the business requirements** instead of worrying about hardware and plugs.

---

## 5. More Users Arrive: The Need for Scalability

### The business requirement

Lots of people are now using your algorithm. (On the whiteboard, many phone icons are drawn, all sending requests to the cloud server.)

```text
  [phone] ──┐
  [phone] ──┤
  [phone] ──┼──►  ( CLOUD: SERVER )   ← too many connections!
  [phone] ──┤
  [phone] ──┘
```

Eventually it reaches a point where **the code running on the machine is not able to handle all of these connections.**

### What can you do? Two solutions

```text
1. BUY BIGGER MACHINE
2. BUY MORE MACHINES
```

### Definition: Scalability

> **Scalability** is the ability to handle more requests, by buying more machines or by buying bigger machines.

The instructor says this is **a very important term to understand well**.

Put simply: **we handle more requests by "throwing more money at the problem."** Either the money buys a bigger machine, or it buys more machines.

---

## 6. The Two Ways to Scale: Vertical and Horizontal

```text
1. BUY BIGGER MACHINE  -  VERTICAL SCALING
2. BUY MORE MACHINES   -  HORIZONTAL SCALING
```

### 6.1 Vertical Scaling (buy a bigger machine)

- Your computer becomes **larger** (more powerful).
- **Therefore it can process requests faster.**
- You still have **one** machine; it is just a stronger one.

Mental picture: making one worker stronger and faster.

### 6.2 Horizontal Scaling (buy more machines)

- You have **multiple machines**.
- A request can **fall on any one of these machines**, and it will be processed there.
- Because you have more machines, the **requests can be randomly distributed** among all the machines you bought.

Mental picture: hiring more workers and splitting the work between them.

### Both are ways to increase scalability

These are the **two mechanisms** by which you can increase the scalability of your system. Remember: scalability = being able to handle more requests.

---

## 7. Vertical vs Horizontal Scaling: The 5 Key Differences

Like any two approaches, we can compare them with their **pros and cons**.

### Whiteboard setup

```text
        HORIZONTAL                    │            VERTICAL
                                      │
   ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐      │     ┌───────────────────────┐
   │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │      │     │                       │
   └───┘ └───┘ └───┘ └───┘ └───┘      │     │       HUGE BOX        │
   (five separate small machines)     │     │                       │
                                      │     └───────────────────────┘
                                      │     (one very large machine)
```

### The comparison table (as written on the whiteboard)

| # | HORIZONTAL | VERTICAL |
|---|---|---|
| 1 | **Load balancing required** | **N/A** (not applicable) |
| 2 | **Resilient** | **Single point of failure** |
| 3 | **Network calls (RPC)** → slow | **Inter-process communication** → fast |
| 4 | **Data inconsistency** | **Consistent** |
| 5 | **Scales well as users increase** | **Hardware limit** |

Now each point in detail.

---

### Point 1: Load Balancing

**Horizontal: Load balancing required**

- In horizontal scaling, requests are distributed among many machines (as said in Section 6.2).
- So you **need some sort of load balancing**: something must decide which request goes to which machine.
- **(Clarification)** *Load balancing* means spreading incoming requests across multiple servers so that no single server is overloaded. The component that does this is usually called a *load balancer*.

**Vertical: N/A**

- If you have a **single machine**, there is **no load to balance** as such. Every request goes to the same box.

---

### Point 2: Failure Handling

**Horizontal: Resilient**

- With lots of machines, **if one machine fails, you can redirect the requests to the other ones.**
- The system keeps working even when a part of it breaks. That property is called being **resilient**.

**Vertical: Single point of failure**

- There is only one machine. **If it goes down, the whole service goes down.**
- A component whose failure takes down the entire system is called a **single point of failure**.

```text
Horizontal:  [1] [2] [X] [4] [5]   → machine 3 crashed, requests go to 1, 2, 4, 5 ✔
Vertical:    [   HUGE BOX  X   ]   → the only machine crashed, service is down ✘
```

---

### Point 3: Communication Speed

**Horizontal: Network calls (RPC) → slow**

- The machines are separate computers, so **all communication between the servers happens over the network.**
- **Network calls are slow**, because they are **I/O** (input/output operations, i.e. data has to travel out of one machine and into another).
- Calls between two services over the network are called **Remote Procedure Calls (RPC)**.
  - **(Clarification)** An RPC is when code on one machine calls a function that actually runs on another machine, over the network.

**Vertical: Inter-process communication (IPC) → fast**

- Everything is on one machine, so different parts of the system talk through **inter-process communication**, which happens **inside the same computer**.
- This is **quite fast** because nothing has to travel over a network.

| | Horizontal | Vertical |
|---|---|---|
| How parts communicate | Over the network (RPC) | Within one machine (IPC) |
| Speed | **Slow** (network I/O) | **Fast** |

---

### Point 4: Data Consistency

**Horizontal: Data inconsistency (a real issue)**

The instructor's example (using the numbered boxes on the whiteboard):

> A transaction where **server 3 sends some data to server 4**, then **4 sends it to 5**, and then **5 sends it to 1**.

```text
   [3] ──data──► [4] ──data──► [5] ──data──► [1]
```

Why this is hard:

- The data is now spread across many machines, so it is **complicated to maintain**.
- Suppose this transaction must be **atomic**.
  - **(Clarification)** *Atomic* means **all-or-nothing**: either every step of the transaction happens, or none of them do. You must never end up with only half of it done.
- To guarantee atomicity across machines, you would have to **lock all the servers**, i.e. **all the databases they are using**, until the transaction completes.
- That is **impractical** (every other request needing those databases would have to wait).

**(Illustration of why this matters)** Imagine 3 → 4 succeeds, but the 4 → 5 step fails. Now servers 3 and 4 have the new data, but 5 and 1 do not. Different machines disagree about what the data is. That is **data inconsistency**.

So what actually happens in practice:

> We usually settle for **some sort of loose transactional guarantee**.

That is exactly **why data consistency is a real issue in horizontal scaling.**

**Vertical: Consistent**

- There is **just one system on which all the data resides.**
- No data is split across machines, so there is nothing to keep in sync. **That is why it is consistent.**

---

### Point 5: Limits on Growth

**Vertical: Hardware limit**

- You **cannot just make the computer bigger and bigger and bigger** to solve the problem forever.
- At some point you hit a **hardware limit**: there is only so much CPU, memory, etc. that one machine can have.

**Horizontal: Scales well as users increase**

- The number of servers you need to throw at the problem is **almost linear** in the number of users added.
- In simple words: as users grow, you keep adding machines roughly in proportion. There is no fixed ceiling like a single machine's hardware limit.

---

### Summary Table with Reasoning

| # | Aspect | Horizontal (more machines) | Vertical (bigger machine) | Why |
|---|---|---|---|---|
| 1 | Load balancing | Required | N/A | Many machines need requests to be distributed; one machine has nothing to balance |
| 2 | Failure | Resilient | Single point of failure | Other machines take over vs. only one machine exists |
| 3 | Communication | Network calls (RPC), slow | Inter-process communication, fast | Network I/O between machines vs. communication inside one machine |
| 4 | Data | Inconsistency | Consistent | Data spread across machines, locking all is impractical vs. all data on one system |
| 5 | Growth | Scales well (servers ≈ linear with users) | Hardware limit | Keep adding machines vs. one machine cannot grow forever |

---

## 8. What Is Used in the Real World? The Hybrid Approach

### Answer: **Both**

We take the **good qualities of each** approach.

**Good qualities taken from vertical scaling:**

1. **Really fast inter-process communication.**
2. **Data being consistent.** The **cache is going to be consistent**, and there are **no dirty reads or dirty writes**, so to speak.
   - **(Clarification)** A *dirty read* is reading data that another operation has changed but not yet finalized (it might still be rolled back). A *dirty write* is overwriting such unfinalized data. Both lead to wrong or inconsistent results.

**Good qualities taken from horizontal scaling:**

1. **It scales well** (vertical has a hardware limit, horizontal does not).
2. **It is resilient**: if one of the servers crashes, somebody else (another server) can come up and handle the work.

### What the hybrid solution actually looks like

> The hybrid solution is **essentially horizontal scaling**, where **each machine is as big a box as possible**, as feasible **money-wise**.

```text
  Hybrid = horizontal scaling of big machines

  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │   BIG   │  │   BIG   │  │   BIG   │   ... add more as users grow
  │   BOX   │  │   BOX   │  │   BOX   │
  └─────────┘  └─────────┘  └─────────┘
```

- **Many machines** → you get resilience and good scaling.
- **Each machine big** → more work can be done inside a single machine (fast IPC, consistent data), so fewer slow network calls and fewer consistency problems.

### How this plays out over time (practical advice)

1. **Initially:** you can **vertically scale as much as you like.** (Simple, fast, consistent.)
2. **Later, when your users start trusting you** (your service grows and becomes important to people): you should **probably go for horizontal scaling.** (Needed for scale and resilience.)

---

## 9. The 3 Major Considerations of System Design, and Trade-offs

When designing a system, always ask:

| Question | Meaning | Which scaling style is naturally good at it |
|---|---|---|
| **Is it scalable?** | Can it handle more requests as users grow? | Horizontal |
| **Is it resilient?** | Does it keep working when a part fails? | Horizontal |
| **Is it consistent?** | Is the data correct and the same everywhere? | Vertical |

### Trade-offs

- With these qualities, **there are always going to be some trade-offs.** You usually cannot get the maximum of all three at once; improving one can hurt another (for example, horizontal scaling gives scalability and resilience but makes consistency harder).

### What system design is (the instructor's definition)

> **System design** is designing a system that **meets the requirements**, where the requirements are such that it is actually **possible, in computer science terms, to build** such a system.

In other words: understand what the business needs, understand the trade-offs, and pick a design that satisfies the needs *and* is realistically buildable.

---

## 10. Key Terms (Quick Glossary)

| Term | Meaning (as used in this lecture) |
|---|---|
| **Algorithm / code** | The useful function you wrote: takes input, gives output |
| **Server** | The computer running your code and serving requests |
| **Client** | The user's device (e.g. a phone) that sends requests |
| **API** (Application Programming Interface) | How your code is exposed over the internet so others can use it |
| **Request** | What the client sends to you |
| **Response** | The output your server returns for a request (one response per request) |
| **Endpoint** | The address clients connect to; must be configured |
| **Cloud** | A set of computers someone provides to you for money (e.g. AWS) |
| **Computation power** | A computer at the cloud provider that can run your algorithm |
| **Remote login** | Logging into a remote computer to put/run your code on it |
| **Scalability** | Ability to handle more requests (by bigger or more machines) |
| **Vertical scaling** | Buying a bigger machine |
| **Horizontal scaling** | Buying more machines |
| **Load balancing** | Distributing requests across multiple machines |
| **Resilient** | System survives the failure of one machine |
| **Single point of failure** | One component whose failure brings down everything |
| **RPC** (Remote Procedure Call) | Network call between services on different machines; slow (I/O) |
| **IPC** (Inter-Process Communication) | Communication within one machine; fast |
| **Atomic transaction** | All-or-nothing operation |
| **Loose transactional guarantee** | Weaker consistency promise used in distributed systems because locking everything is impractical |
| **Dirty read / dirty write** | Reading / overwriting data that is not yet finalized |
| **Hardware limit** | The maximum size one machine can be |
| **Hybrid scaling** | Horizontal scaling where each machine is as big as affordable |

---

## 11. Revision and Interview Questions

**Q1. Why should a service be hosted on the cloud instead of your own desktop?**
Running it yourself means handling the database setup, endpoint configuration, and failures like power loss or someone pulling the plug. Paying customers cannot tolerate downtime. A cloud provider (e.g. AWS) takes care of configuration, settings, and reliability to a large extent, so you can focus on business requirements.

**Q2. Is the cloud fundamentally different from a desktop?**
No. The cloud is just a set of computers someone else owns and rents to you. You access them via remote login and run your code there.

**Q3. What is scalability?**
The ability of a system to handle more requests, achieved by buying bigger machines or more machines.

**Q4. What is the difference between vertical and horizontal scaling?**
Vertical = buy a bigger machine (processes requests faster). Horizontal = buy more machines (requests distributed among them).

**Q5. Give the 5 differences between horizontal and vertical scaling.**
(1) Load balancing: required vs N/A. (2) Resilient vs single point of failure. (3) Network calls/RPC (slow) vs inter-process communication (fast). (4) Data inconsistency vs consistent. (5) Scales well as users increase vs hardware limit.

**Q6. Why is data consistency a problem in horizontal scaling?**
Data is spread over many servers. For an atomic transaction passing through several servers (e.g. 3 → 4 → 5 → 1), you would need to lock all involved servers/databases, which is impractical. So systems use loose transactional guarantees, which can lead to inconsistency.

**Q7. Why are network calls slower than inter-process communication?**
Network calls are I/O: data must travel between separate machines. IPC happens inside a single machine.

**Q8. What is used in the real world?**
Both, as a hybrid: horizontal scaling where each machine is as big as feasible money-wise. From vertical you keep fast IPC and consistency (consistent cache, no dirty reads/writes); from horizontal you get good scaling and resilience.

**Q9. When should you move from vertical to horizontal scaling?**
Vertically scale as much as you like initially; move to horizontal scaling later, once your service grows and users start trusting and depending on you.

**Q10. What are the three major considerations when designing a system?**
Is it scalable? Is it resilient? Is it consistent? There are always trade-offs between them.

**Q11. What is system design?**
Designing a system that meets the requirements, where those requirements are actually possible to build in computer science terms.
