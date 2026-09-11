# System Design — Lecture 6: Monolith vs Microservices

> **About these notes:** Built from the full lecture transcript and all on-screen whiteboard content and text overlays. Everything the instructor said or drew is included. Anything **not** from the lecture is clearly labelled **[Supplementary]**: these are short explanations of terms the lecture uses without defining, plus corrections where a statement in the lecture is oversimplified. Section 11 collects all corrections and clarifications in one place.

---

## Table of Contents

1. [What This Lecture Is About](#1-what-this-lecture-is-about)
2. [The Common (Incorrect) Mental Picture](#2-the-common-incorrect-mental-picture)
3. [Clearing the Misconceptions](#3-clearing-the-misconceptions)
4. [Monolith Architecture — Advantages](#4-monolith-architecture--advantages)
5. [Monolith Architecture — Disadvantages](#5-monolith-architecture--disadvantages)
6. [Microservice Architecture — Advantages](#6-microservice-architecture--advantages)
7. [Microservice Architecture — Disadvantages](#7-microservice-architecture--disadvantages)
8. [Side-by-Side Comparison](#8-side-by-side-comparison)
9. [How to Handle This in a System Design Interview](#9-how-to-handle-this-in-a-system-design-interview)
10. [Real-World Examples](#10-real-world-examples)
11. [Corrections & Clarifications](#11-corrections--clarifications)
12. [Supplementary Concepts (Gap-Filling)](#12-supplementary-concepts-gap-filling)
13. [Exam & Interview Traps](#13-exam--interview-traps)
14. [Quick Reference](#14-quick-reference)
15. [One-Page Revision Sheet](#15-one-page-revision-sheet)

---

## 1. What This Lecture Is About

**The central question:** When building a software system, should you choose a **monolith** architecture or a **microservice** architecture?

This topic is intensely debated in software engineering circles and comes up very often in **system design interviews**.

**Flow of the lecture:**

1. The picture most people have in their heads of monoliths and microservices.
2. Why that picture is wrong (two misconceptions).
3. Advantages and disadvantages of monoliths.
4. Advantages and disadvantages of microservices.
5. What to actually do in a system design interview.
6. Real companies using each approach.

---

## 2. The Common (Incorrect) Mental Picture

The instructor first draws what "most people think" these architectures look like, and then spends the next part of the lecture showing why this picture is wrong.

### 2.1 Monolith — as most people imagine it

The belief:
- A monolith is a **huge system**.
- It is **just one machine** running the **entire system**.
- **All clients** connect to this **single machine**.
- That machine talks to a single database.

**Whiteboard (00:00–00:51), top diagram:**

```
   [Client]    [Client]    [Client]
       \           |           /
        \          |          /
         v         v         v
      +-----------------------------+
      |          MONOLITH           |   <- "one giant machine runs everything"
      +-----------------------------+
                   |
                   v
              ( Database )
```

### 2.2 Microservices — as most people imagine it

The belief:
- Microservices are **"tiny, tiny dots"**.
- Each one runs **just one function on one machine**.
- Each has its own **tiny database**.
- They are all **interconnected** with each other in a web.
- **Clients connect directly to different microservices.**

**Whiteboard (00:00–00:51), bottom diagram — a dense mesh:**

```
   [Client]          [Client]           [Client]
      |   \             |    \             |
      v    \            v     \            v
     (s)----\----------(s)-----\----------(s)
      | \    \        /  | \    \        / |
      |  \    v      /   |  \    v      /  |
    [db]  (s)-----(s)   [db] (s)-----(s)  [db]
           |       |          |       |
         [db]    [db]       [db]    [db]

   (s)  = tiny service running a single function
   [db] = tiny database attached to it
   Clients talk to services directly; services form a tangled web of calls.
```

The instructor ends this part with: *"...is what most people think."* In other words, **both pictures are misconceptions.**

---

## 3. Clearing the Misconceptions

### 3.1 Misconception #1: "A monolith is one huge machine running the whole system"

**This is false.** A monolith does **not** need to be single in terms of the **number of machines** it runs on.

- You can run the **same monolith** on **multiple machines**.
- Clients can connect to any of these machines.
- These machines, in turn, connect to the database, or even to **more than one database**.
- Therefore you **can horizontally scale (scale out) a monolith**. In the instructor's words, *"there's nothing special about it."*

**Whiteboard (00:52–01:14), green additions to the monolith diagram:** the instructor adds two more boxes labelled `M` (more monolith machines), connects clients to them, and connects them to the main database and to additional database cylinders.

```
   [Client]     [Client]     [Client]     [Client]
       |            |            |            |
       v            v            v            v
  +----------+  +----------+  +----------+
  | MONOLITH |  |    M     |  |    M     |    <- identical copies of the SAME monolith
  +----------+  +----------+  +----------+
        \          |     \          \
         \         |      \          \
          v        v       v          v
         ( Main Database )  ( Extra DB )  ( Extra DB )
```

**Why this matters:** "Monolith" is **not** about hardware size or machine count. It describes how the **code/system is organised**: all the functionality lives together in one service. How many copies of that service you run is a separate decision.

> **Intuition:** Think of a monolith like a single restaurant menu where one kitchen cooks everything. Opening three branches of that restaurant (three identical kitchens, each cooking the whole menu) doesn't turn it into three specialised restaurants. It's still one "everything-kitchen" design, just replicated.

---

### 3.2 Misconception #2: "Microservices are tiny"

The instructor's key line: **"There's nothing micro about a microservice."**

**What a microservice actually is:**

- A microservice is **a single business unit**.
- **All data and all functions** relevant to that business unit are put into **that one service**.
- You split a service into smaller pieces **only if** it has many concerns that can genuinely be separated out. Splitting is justified by *separation of concerns*, not by a goal of making things small.
- It is **not** "tiny machines which interact with each other all the time."
- The **number** of microservices can be small. There might be **just three microservices in your entire architecture**, depending on what your system is.
- Each microservice **usually talks to its own dedicated database**.

**Example business units** (from the later diagram in the lecture): **Profiles**, **Analytics**, **Chat**. Everything related to chat, meaning its logic and its data, belongs to the Chat service. Everything related to profiles belongs to the Profiles service, and so on.

---

### 3.3 Clients usually talk to a Gateway, not directly to microservices

The second part of the misconception was that clients talk directly to various microservices. In practice:

- The client **may not be talking to the microservice directly**.
- Clients connect to a **gateway**.
- The **gateway talks to the microservices internally**.

**Whiteboard (01:40–02:04), green additions to the microservices diagram:**

```
   [Client]     [Client]     [Client]
       \            |            /
        v           v           v
      +-------------------------------+
      |            GATEWAY            |   <- single entry point for clients
      +-------------------------------+
          /             |            \
         v              v             v
   +-----------+  +-----------+  +-----------+
   | Service 1 |  | Service 2 |  | Service 3 |   <- only 3 services; each is a business unit
   +-----------+  +-----------+  +-----------+
         |              |              |
         v              v              v
      ( DB 1 )       ( DB 2 )       ( DB 3 )     <- each service has its OWN dedicated DB
```

> **[Supplementary]** An **API Gateway** is the single front door of a microservice system. It receives every client request and **routes** it to the correct internal service (e.g., `/chat/...` → Chat service). Clients don't need to know how many services exist or where they live. See §12.3.

---

### 3.4 Corrected Definitions — Summary

| | **Common belief (wrong)** | **Reality (as per the lecture)** |
|---|---|---|
| **Monolith** | One huge machine running the entire system; all clients hit that one machine | All functionality in one service. The same monolith can run on **many machines** (horizontal scaling) and can use **one or more** databases |
| **Microservice** | A tiny service running a single function on its own machine, part of a dense mesh; clients hit services directly | A **single business unit** containing all its related data and functions. There may be only a **few** of them. Each usually has its **own dedicated DB**. Clients typically go through a **gateway** |

---

## 4. Monolith Architecture — Advantages

**Whiteboard (02:12–02:26): scaled monolith**

```
   [Client]      [Client]      [Client]
       |             |             |
       v             v             v
   +-------+     +-------+     +-------+
   |  MS1  |     |  MS2  |     |  MS3  |      MS = Monolith Server
   +-------+     +-------+     +-------+      (NOT "microservice" — see Trap #6)
        \            |            /
         \           |           /
          v          v          v
           ( Shared Database )
```

When put under **a lot of load**, the monolith **scales out** exactly like this: you add more servers, each running the full monolith, all sharing the database. With that established, here are its advantages.

---

### 4.1 Good for Small Teams
*(On-screen overlay at 02:26: **"Good for small teams"**)*

- If you have a **small, cohesive team**, you may not be able to afford:
  - the **time** required to break the system into microservices, and
  - the **interactions** (coordination and communication overhead) that multiple services demand.
- **Instructor's rule:** *If your team is cohesive, go for a monolith architecture.*

**Why:** Microservices bring coordination costs: agreeing on how services talk to each other, who owns what, how to deploy each piece. A small team that already works closely together gets little benefit from these boundaries and pays all of their cost.

---

### 4.2 Less Complex (Fewer Moving Parts)
*(On-screen overlay at 02:42: **"Less Complex"**)*

- There are **fewer moving parts** in this architecture.
- You don't have to wonder:
  - **"How do I break this into pieces?"**
  - **"Once they are broken into pieces, how do I maintain different servers at different times?"**
- **Deployments are, in some way, easy**, because **everything is the same**: one codebase, one type of server, one thing to deploy.

> Note: In §5.2 the instructor also lists "Complicated Deployments" as a *disadvantage*. These two statements are about different aspects of deployment. See §5.4 for how they fit together.

---

### 4.3 Less Duplication
*(On-screen overlay at 03:12: **"Less Duplication"**)*

- Every system has supporting code, such as code for:
  - **setting up tests**,
  - **setting up connections** (e.g., to databases),
  - **setting up various other things** in the system.
- In a monolith, **this code does not need to be duplicated for every service you create**. It's all in one service, written once, shared by everything.

**Flip side (implied):** In microservices, each service is a separate application, so each needs its own test setup, connection setup, configuration, etc.

---

### 4.4 Procedure Call Is Faster
*(On-screen overlay at 03:20: **"Procedure call is faster"**)*

- In a monolith, when one part of the code needs another part, it makes a **local call**. Both pieces of code are **in the same box**.
- You are **not making any calls over the network**. It is **not an RPC** (Remote Procedure Call).
- Since all the logic and code sit in the same box, **it runs faster**. *"You just need to make a local call."*

**Why local calls are faster:** A local function call happens inside the same process's memory. A remote call must package the data, send it across the network, wait for the other machine to process it, and receive the reply. See §12.4 for a comparison.

---

## 5. Monolith Architecture — Disadvantages

*("Now let me come to the fun bit, which is disadvantages.")*

### 5.1 More Context Required
*(On-screen overlay at 03:46: **"More context required"**)*

- The **biggest disadvantage**, according to the instructor.
- When a **new member joins the team**, they need **a lot of context** about what they're developing.
- If you hand them a monolith containing **all the logic**, they must go through and **understand the whole system**, holding all of it in their mind, before they can safely make changes.

**Why:** In a monolith there are no hard boundaries between features. A change in one place might affect code anywhere else, so a developer can't safely work on one part while ignoring the rest.

---

### 5.2 Complicated Deployments (and Complicated Tests)
*(On-screen overlay at 03:58: **"Complicated Deployments"**)*

**Deployments:**
- **Any change in the code**, whichever part you touch, **requires a new deployment** of the whole monolith.
- As a result, your code will be **deployed very frequently**.
- **Every deployment must be monitored** to make sure the system is still working properly, because every deployment touches the entire system.

**Tests:**
- **Tests are also more complicated**, because *"everything is touching everything."*
- The code is **not decoupled** the way you would like. A change in one module can break a completely different module, so testing one change can mean testing much more of the system.

---

### 5.3 Single Point of Failure
*(On-screen overlay at 04:26: **"Single Point of Failure!"**)*

- There is **too much responsibility on each server**. Every server does *everything*.
- If there's **a mistake (a bug) that makes the server crash**, then **all of the servers will crash**, since all of them run the same code, and **the whole system collapses**.

**The alternative the instructor draws (04:30–04:53): separate `Profile` and `Data` boxes**

Suppose instead you had:
- one thing serving **profiles**, and
- another thing serving **some other data**.

Some clients connect to the Profile server and some to the Data server.

```
   [Client A]                    [Client B]
       |                             |
       v                             v
  +-----------+                 +-----------+
  |  Profile  |   X  CRASHED    |   Data    |   ✓ STILL WORKING
  +-----------+                 +-----------+
       |                             |
  restart ONLY this             keeps serving users
```

- The Profile one **fails**; the Data one **succeeds**.
- You get **partial success** in the system instead of total failure.
- You **only need to restart the failing piece**.
- With the monolith, you instead **needed to restart all of the servers with the correct code**.

#### Worked Scenario — A Buggy Profile Feature

| Step | Monolith (MS1, MS2, MS3 all run all code) | Separated (Profile server + Data server) |
|---|---|---|
| 1. Bug is shipped | Bug in profile code is deployed to **MS1, MS2, MS3** (everyone runs the same code) | Bug is deployed only to the **Profile** server |
| 2. Bug is triggered | Profile requests crash whichever server handles them. Since every server has the bug, **each one goes down** | Profile server crashes |
| 3. Effect on other features | Data features are **also down**, since they lived in the crashed servers | Data server is **unaffected**, users can still get data |
| 4. System state | **Total outage** | **Partial success** |
| 5. Recovery | Fix the code, **redeploy and restart ALL servers** | Fix and **restart only the Profile server** |

> **Clarification:** "Single point of failure" usually means one component whose failure takes everything down. A monolith running on 3 servers *is* protected against **one machine dying**. What it is **not** protected against is a **bug**, because the bug exists on every copy. So the single point of failure here is the **shared code**, not a single machine. See §11, C2.

---

### 5.4 Reconciling "Deployments Are Easy" (§4.2) vs "Complicated Deployments" (§5.2)

The lecture states both. They describe **different dimensions** of deployment:

| Aspect | Why it's **easier** in a monolith | Why it's **harder** in a monolith |
|---|---|---|
| **What** you deploy | Only one kind of thing: one codebase, identical servers, one process to set up | — |
| **How often** & **how risky** | — | *Every* change, anywhere, triggers a redeploy of the *whole* system, so deploys are frequent, and each one risks the entire system and must be monitored |

**Short version:** Monolith deployments are **simple in mechanics** but **heavy in frequency and blast radius**.

---

## 6. Microservice Architecture — Advantages

**Whiteboard (04:55–05:00):**

```
   [Client]     [Client]     [Client]
       \            |            /
        v           v           v
      +-------------------------------+
      |            GATEWAY            |
      +-------------------------------+
          /             |             \
         v              v              v
   +-----------+  +-------------+  +-----------+
   |  Profiles |  |  Analytics  |  |   Chat    |
   +-----------+  +-------------+  +-----------+
         |              |                |
         v              v                v
   ( Profiles DB )  ( Analytics DB )  ( Chat DB )
```

---

### 6.1 Scalability (Easier to Scale and Design)
*(On-screen overlay at 05:00: **"Scalability"**)*

- The *"most obvious advantage"*: it's **easier to scale**.
- **Why (instructor's reasoning):** You can look at the entire system as **a suite, a set of services**, where:
  - each service is **concerned only with its own data**, and
  - services **interact with each other**.
- Because of this, *"it's easier to design the system in that way."*

**Intuition:** When each service owns one business area and its data, you can think about, design, and grow each piece independently without having to reason about the entire system at once. The concrete "add machines to just one service" example comes in §6.4.

---

### 6.2 Easier for New Team Members
*(On-screen overlay at 05:22: **"Easier for new team members"**)*

- When a **new developer** joins, you can assign them **a task concerning a particular service**.
- They only need **the context of that one service**, not the whole monolith.
- This directly solves the monolith's "More context required" problem (§5.1).

---

### 6.3 Working in Parallel
*(On-screen overlay at 05:32: **"Working in Parallel"**)*

- **Parallel development is easy** because there is **less dependency between teams**.
- Example from the lecture: the **Chat developers** depend less on the **Analytics developers**, so both can develop **at the same time**.
- **In a monolith:** maybe one function calls another function, and that other function is being changed. This creates **tight coupling**, and the instructor stresses it is coupling **not just in code, but also in developer time**.

**What "coupling in developer time" means (elaboration of the lecture's example):**

```
MONOLITH (tight coupling)
  Chat code ──directly calls──> analyticsFunction()
                                     ^
                                     |
                      Analytics dev is changing this function
  → Chat dev's code may break, or Chat dev must WAIT / coordinate
    → one team's schedule now depends on the other team's schedule

MICROSERVICES (looser coupling)
  Chat Service ──talks through defined interface──> Analytics Service
  → Each team changes its internals freely as long as the interface stays the same
    → Both teams work simultaneously
```

---

### 6.4 Easier to Reason About (Targeted Scaling)
*(On-screen overlay at 05:58: **"Easier to reason about"**)*

- When you deploy a service, **fewer parts are hidden**. Each deployed service clearly corresponds to one business function, so you can **see** what's happening in it.
- **Example:** If you see **a lot of load on the Chat server**, you can **easily scale it out** by putting **more machines for just the chat code**.
- **With a monolith**, it's **more difficult to tell what is being used a lot and what is being used less**, because all features run mixed together inside the same servers. So you are more likely to just **add more servers directly**, each a full copy of everything.
- Microservices allow **a more streamlined approach** to the problem.

#### Worked Scenario — Chat Traffic Spikes

| Step | Monolith | Microservices |
|---|---|---|
| 1. Load increases | Servers MS1–MS3 get busy | Chat service machines get busy |
| 2. Diagnose | Hard to say whether chat, profiles, or analytics is causing it; everything runs in the same process | Clearly visible: **Chat** service is under load, Profiles and Analytics are fine |
| 3. Respond | Add more **full monolith servers** (each also carries profile + analytics code that didn't need scaling) | Add more machines **only for Chat** |
| 4. Result | Works, but wasteful and less precise | **Streamlined**: resources go exactly where needed |

> **Note on the on-screen labels:** The spoken explanation under "Scalability" (§6.1) is mostly about *modular design*, while the concrete *independent scaling* example is spoken under "Easier to reason about" (§6.4). Treat the two points together: **visibility into each service → scale each service independently.**

---

## 7. Microservice Architecture — Disadvantages

### 7.1 Not Easy to Design — Risk of Over-Splitting

- Microservices are **not easy to design**.
- **Example from the lecture:** the **Chat** functionality might get broken into **far more parts than required**.
- This is exactly the "tiny dots" misconception from §2.2 turning into a real design mistake.

### 7.2 Red Flag: A Service That Only Talks to One Other Service

The instructor gives a **practical indicator** that something *shouldn't* be a separate microservice:

> **If a service is talking only to one other service**, e.g., Service 1 is **just talking to Service 2 all the time**, that implies they **should probably have been a single service**. Then the **RPC** (network call) between them could be converted into a **normal function call**.

**Whiteboard (06:36–06:55):** The instructor draws `S1 --call--> S2` and then draws a single container box around both to show them merged.

```
BEFORE (over-split)                          AFTER (merged)

 +------+   RPC over network   +------+      +---------------------------------+
 |  S1  | -------------------> |  S2  |      |  +------+  local   +------+     |
 +------+     (all the time)   +------+  =>  |  |  S1  | -------> |  S2  |     |
                                             |  +------+ function +------+     |
 S1 exists only to call S2                   |            call                 |
                                             +---------------------------------+
                                                     ONE service
```

**Why merging helps:** You get back the monolith advantage from §4.4 (local procedure calls are faster, no network hop) and remove an unnecessary moving part, without losing any real separation, since the two pieces were never truly independent anyway.

### 7.3 Needs Skilled Architects
*(On-screen overlay at 06:56: **"Needs skilled Architects"**)*

- The instructor calls this *"the only disadvantage"*: a microservice architecture **needs a smart architect** to design it well, deciding correctly where service boundaries go.

> ⚠️ **"Only disadvantage" is an overstatement.** By the lecture's own monolith advantages (§4), microservices carry the opposite costs: more moving parts, duplicated setup code, slower network calls, and heavier coordination for small teams. Interviewers expect you to know these. See §11, C1.

---

## 8. Side-by-Side Comparison

Every monolith advantage roughly mirrors a microservices disadvantage, and vice versa. Cells marked *(implied)* are the logical flip side of a point the instructor made about the other architecture.

| Dimension | Monolith | Microservices |
|---|---|---|
| **Definition** | All functionality in one service | Each service = one business unit with its own data & functions |
| **Can scale horizontally?** | Yes: run many identical copies | Yes: scale each service independently |
| **Scaling precision** | Coarse: hard to tell what's heavily used, so you add full servers | Targeted: add machines to just the loaded service (e.g., Chat) |
| **Team size fit** | Small, cohesive teams | Larger teams working in parallel *(implied)* |
| **Complexity / moving parts** | Fewer moving parts | More pieces to break apart and maintain *(implied)* |
| **Setup code (tests, connections)** | Written once, shared | Duplicated per service *(implied)* |
| **Internal calls** | Local procedure calls: faster | RPC over the network: slower *(implied)* |
| **Context for new developers** | Must understand the whole system | Only needs one service's context |
| **Deployments** | Simple mechanics, but any change redeploys everything, frequently, with monitoring each time | Deploy only the changed service *(implied)* |
| **Testing** | Complicated: everything touches everything | Services are decoupled *(implied)* |
| **Parallel development** | Tight coupling in code and developer time | Teams (Chat vs Analytics) work simultaneously |
| **Failure impact** | A crashing bug takes down all servers: total collapse | Partial success: restart only the failing service |
| **Design difficulty** | Straightforward | Hard; risk of over-splitting; needs skilled architects |
| **Typical DB setup** | Shared database (possibly more than one) | Dedicated database per service |
| **Client entry point** | Clients reach the monolith servers | Clients usually go through a gateway |
| **Real-world example** | Stack Overflow | Google, Facebook, and many others |

---

## 9. How to Handle This in a System Design Interview

**What the instructor says:**

1. In a system design interview, you **may need to justify** why you chose a microservice architecture.
2. **About 90% of the time** (instructor's rough estimate), the interview problem will be a **large system**.
3. For a **larger system**, a microservice architecture is **more often than not better**.
4. So **go for microservices by default**.
5. If asked to justify the decision, **use the points from this lecture** (scalability, independent scaling, parallel development, easier onboarding, fault isolation).
6. **If your justifications don't satisfy the interviewer**, they may be **hinting that you should go for a monolith**, so be ready to switch and explain why.

**Decision flow:**

```
          Interview problem given
                    |
                    v
        Is it a large system? (usually yes)
             /                 \
           YES                  NO / hints of small team, small scope
            |                        |
            v                        v
   Default: MICROSERVICES      Consider MONOLITH
            |                   (small cohesive team, fewer moving parts,
            v                    less duplication, faster local calls)
   Interviewer asks "why?"
            |
            v
   Justify with lecture points
            |
     Accepted? ---- NO ----> Take it as a hint → reconsider monolith
            |
           YES → continue design
```

**Example justification** (built only from lecture points):
> "Since this is a large system, I'll split it by business unit, say Profiles, Chat, and Analytics, each with its own database behind a gateway. That lets us scale Chat independently when its load spikes, lets teams develop in parallel without tight coupling, makes onboarding easier since a developer only needs one service's context, and means a crashing bug in one service gives partial success instead of taking the whole system down. I'll avoid over-splitting: if two services only ever talk to each other, I'll merge them."

---

## 10. Real-World Examples

| Architecture | Company / System | Lecture's point |
|---|---|---|
| **Monolith** | **Stack Overflow** | A *very successful* system built as a monolith, proof that a monolith can work at scale (ties back to Misconception #1) |
| **Microservices** | **Google, Facebook**, "and many, many companies" | Microservices are used extensively in large companies |

> **[Supplementary nuance]** Large companies rarely use a pure form of either. For example, Facebook's main web application has historically been a large, monolithic codebase that sits in front of many separate backend services. In an interview, it's accurate to say big companies use a mix, with microservices heavily used.

The instructor closes by inviting discussion on which is better, which itself signals that there's **no universally correct answer**; it depends on the system and the team.

---

## 11. Corrections & Clarifications

### C1. "That's the only disadvantage" of microservices — Overstated
The instructor lists only design difficulty / needing skilled architects. But the lecture's **own** monolith advantages imply more microservice disadvantages:

| Monolith advantage (from lecture) | Corresponding microservice disadvantage |
|---|---|
| Good for small teams | Coordination overhead; a small team may not be able to afford the time and interactions |
| Less complex | More moving parts: many services to deploy, monitor, and maintain |
| Less duplication | Setup code (tests, connections, config) repeated per service |
| Procedure call is faster | Service-to-service calls go over the network (RPC): slower, and they can fail or time out |

**[Supplementary]** Two further commonly-cited costs, which follow from "each service has its own database":
- **Data consistency:** An operation that touches data in two services' databases can't be handled as one simple database transaction.
- **Debugging:** A single user request may pass through several services, so tracing a problem is harder than stepping through one codebase.

### C2. "Single Point of Failure" when the monolith has multiple servers
Running MS1, MS2, MS3 **does** protect against one **machine** failing. The lecture's point is about a **bug**: every server runs the same code, so a crashing bug hits all of them. The single point of failure is the **shared codebase**, not one physical machine.

**[Supplementary]** Microservices' "partial success" works in the lecture's example because Profile and Data are **independent**. If Service A must call Service B to answer a request and B is down, A's requests fail too. Isolation depends on how dependent the services are.

### C3. Deployments listed as both an advantage and a disadvantage of monoliths
Not a real contradiction: deployments are **mechanically simple** (one identical thing everywhere) but **frequent and high-stakes** (every change redeploys the whole system). See §5.4.

### C4. On-screen label vs narration mismatch (microservice advantages)
Under "**Scalability**", the narration mainly describes *modular design* (a suite of services, each concerned with its own data). The concrete *independent scaling* example (more machines for Chat) is narrated under "**Easier to reason about**". Study them as one connected idea. See §6.4 note.

### C5. The "S1 only talks to S2" rule is a heuristic, not an absolute law
The instructor presents it as a *"good indicator."* The real warning sign is **constant, tightly coupled communication**, where S1 exists mainly to call S2. **[Supplementary]** A service that happens to be called by only one other service can still justifiably be separate if it has a very different load pattern or needs to fail independently.

### C6. On-screen "MS1, MS2, MS3" means Monolith Servers
In the 02:12 diagram, `MS1–MS3` are **copies of the monolith**, not microservices. Easy to misread since "MS" is also a common abbreviation for microservice.

### C7. "Procedure call is faster" — scope of the claim
The on-screen text says "Procedure call is faster"; the narration contrasts this with an **RPC (Remote Procedure Call)**. The claim is about **internal calls between parts of the code**: local calls beat network calls. It does not mean a monolith is automatically faster overall in every situation.

### C8. "90% of the time" is the instructor's estimate
It's a rough rule of thumb for how often interview problems involve large systems, not a measured statistic. "Microservices by default" is a starting stance for large-system interviews, not a universal rule.

---

## 12. Supplementary Concepts (Gap-Filling)

> **Everything in this section is [Supplementary].** These terms appear in the lecture but aren't explained there.

### 12.1 Horizontal vs Vertical Scaling
The lecture says a monolith can "horizontally scale" / "scale out."

| | **Vertical scaling (scale up)** | **Horizontal scaling (scale out)** |
|---|---|---|
| What you do | Make one machine bigger (more CPU, RAM) | Add more machines running the same software |
| Limit | Hard ceiling: machines only get so big | Can keep adding machines |
| In the lecture | The misconception picture: "one huge machine" | MS1, MS2, MS3 running copies of the monolith |

### 12.2 Load Balancer
In the lecture's scaled-monolith diagram, clients are drawn connecting directly to MS1–MS3. In practice, a **load balancer** usually sits in front and **distributes incoming requests** across the servers, so no single copy gets overloaded.

### 12.3 API Gateway
The **single entry point** for clients in a microservice system (§3.3). It receives requests and **routes** each one to the correct internal service. Clients don't need to know how many services exist or where they run, so the internal structure can change without breaking clients.

### 12.4 RPC vs Local Procedure Call

| | **Local procedure call** (monolith) | **Remote Procedure Call — RPC** (microservices) |
|---|---|---|
| Where the called code runs | Same process, same machine | Different service, usually a different machine |
| What happens | Direct jump to the function in memory | Data is packaged, sent over the network, processed remotely, and a response is sent back |
| Speed | Very fast | Much slower (network travel + packaging) |
| Can the call itself fail? | Only if the code itself fails | Yes: network issues, timeouts, the other service being down |

### 12.5 Database per Service
In the lecture's microservice diagrams, each service has its **own dedicated database**. This lets a service **own its data**: other services don't reach into its tables, which keeps services decoupled and lets each be changed or scaled independently. The trade-off is the data consistency issue mentioned in C1.

### 12.6 Coupling / Decoupling
- **Tight coupling:** Parts depend heavily on each other's internal details, so changing one forces changes in others. The lecture's monolith example: a function calling another function that's being changed.
- **Decoupling:** Parts interact only through stable, well-defined boundaries, so each can change independently. This is what enables "Working in Parallel."

### 12.7 Single Point of Failure (SPOF)
A component whose failure brings down the whole system. See C2 for why, in a replicated monolith, the SPOF is the shared code rather than a machine.

---

## 13. Exam & Interview Traps

**Trap 1: "A monolith runs on a single machine."**
❌ False. A monolith can run on many machines and be horizontally scaled. "Monolith" is about how functionality is organised, not machine count.

**Trap 2: "Microservices must be tiny / one function each."**
❌ False. *"There's nothing micro about a microservice."* A microservice is a **single business unit** containing all related data and functions.

**Trap 3: "A microservice architecture must have lots of services."**
❌ False. You might have **just three** microservices in the entire system.

**Trap 4: "Clients call microservices directly."**
Usually not. Clients typically talk to a **gateway**, which routes to services internally.

**Trap 5: "Monoliths can't scale."**
❌ False. They scale out by adding servers. What's harder is **targeted** scaling, since it's hard to tell which part is under load, so you end up adding full copies.

**Trap 6: Reading `MS1, MS2, MS3` as microservices.**
In this lecture, those are **monolith servers**.

**Trap 7: "Running 3 monolith servers removes the single point of failure."**
Only for hardware failure. A **crashing bug** is present on all copies, so it can take the whole system down. With separate services, you get **partial success** and restart only the broken one.

**Trap 8: "Design difficulty is the only downside of microservices."**
Oversimplified. Also: more moving parts, duplicated setup code, slower and failure-prone network calls, coordination overhead (and data consistency across databases).

**Trap 9: Over-splitting.**
If Service 1 talks **only** to Service 2 all the time, they probably should be **one service**, turning the RPC into a local function call.

**Trap 10: "Monolith deployments are easy" vs "complicated."**
Both are true for different reasons: simple mechanics vs. frequent, whole-system redeploys needing monitoring.

**Trap 11: Always insisting on microservices in an interview.**
Default to microservices for large systems, but if your justification doesn't satisfy the interviewer, **treat it as a hint** to consider a monolith.

**Trap 12: "Microservices are always better — big companies use them."**
Stack Overflow is a **very successful monolith**. The right choice depends on team size and system needs.

---

## 14. Quick Reference

### Monolith

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Scales out by adding identical servers | More context required for new members |
| Good for small, cohesive teams | Complicated deployments: any change redeploys everything, frequently, with monitoring |
| Less complex: fewer moving parts, simple deploys | Complicated tests: everything touches everything |
| Less duplication: shared setup code | Single point of failure: a crashing bug takes down all servers |
| Local procedure calls are faster than RPC | Hard to tell which part is heavily used, so scaling is coarse |

### Microservices

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Scalability: suite of services, each with its own data | Hard to design: risk of over-splitting (e.g., Chat split into too many parts) |
| Easier for new team members: one service's context | Needs skilled architects |
| Working in parallel: less coupling in code & developer time | *(Implied / supplementary: more moving parts, duplicated setup, slower RPCs, cross-DB consistency)* |
| Easier to reason about: see which service is loaded, scale just that | |
| Partial success when one service crashes | |

### Key Heuristics
- **Small cohesive team** → lean monolith.
- **Large system (most interviews)** → default microservices.
- **S1 only ever talks to S2** → merge into one service.
- **Interviewer rejects your justification** → hint toward monolith.

---

## 15. One-Page Revision Sheet

```
MONOLITH vs MICROSERVICES — SYSTEM DESIGN LEC 6
=======================================================================

MISCONCEPTIONS
  ✗ Monolith = one giant machine
    ✓ Same monolith can run on MANY machines → horizontal scaling; 1+ DBs
  ✗ Microservice = tiny, one function, dense mesh, clients call directly
    ✓ "Nothing micro about a microservice"
    ✓ Microservice = SINGLE BUSINESS UNIT (all its data + functions)
    ✓ Can be just 3 services total; each usually has its OWN DB
    ✓ Clients → GATEWAY → services

MONOLITH  (diagram: Clients → MS1/MS2/MS3 → Shared DB; MS = monolith server)
  + Good for small teams       (cohesive team can't afford split time/interactions)
  + Less complex               (fewer moving parts; deploy one identical thing)
  + Less duplication           (test/connection/setup code written once)
  + Procedure call is faster   (local call, same box, no RPC/network)
  - More context required      (new dev must understand whole system)  ← biggest
  - Complicated deployments    (ANY change → redeploy all, often, monitor each time)
  - Complicated tests          (everything touches everything, not decoupled)
  - Single point of failure    (crashing bug on all copies → whole system collapses;
                                separate Profile/Data → partial success, restart one)

MICROSERVICES  (diagram: Clients → GATEWAY → Profiles | Analytics | Chat → own DBs)
  + Scalability                (suite of services, each concerned with own data)
  + Easier for new members     (only one service's context)
  + Working in parallel        (Chat & Analytics devs independent;
                                monolith = tight coupling in code AND dev time)
  + Easier to reason about     (see Chat is loaded → add machines only for Chat;
                                monolith → hard to tell, add full servers)
  - Hard to design             (may over-split, e.g., Chat into too many parts)
  - RED FLAG: S1 talks only to S2 → merge; RPC → normal function call
  - Needs skilled architects   (lecture: "only disadvantage" — OVERSTATED:
                                also more moving parts, duplication, slow RPC)

RECONCILE: Monolith deploys = simple mechanics BUT frequent + whole-system risk
CLARIFY:   Replicas stop hardware SPOF, NOT bug SPOF (same code everywhere)

INTERVIEW
  • ~90% of problems are large systems → DEFAULT to microservices
  • Be ready to JUSTIFY with the points above
  • Justification rejected? → interviewer hinting MONOLITH

EXAMPLES
  • Monolith:       Stack Overflow (very successful)
  • Microservices:  Google, Facebook, many others (big companies often mix)
=======================================================================
```
