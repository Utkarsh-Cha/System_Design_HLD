# System Design — Lecture 3: Consistent Hashing (Part 1: The Problem It Solves)

*Instructor: Gaurav Sen (GKCS)*

---

## 0. What this lecture covers (read this first)

This lecture does **not** yet teach how consistent hashing works internally. It builds the **motivation**: it shows you the simple, "normal" way of spreading requests across servers, and then shows exactly **why that simple way breaks** when you add a server. The lecture ends right at the point where consistent hashing is introduced as the fix.

The story goes like this:

1. One server running your program → it gets too many users.
2. You add more servers → now you must decide which server gets which request (**load balancing**).
3. Simple solution: `hash(request ID) % N` → works well and spreads load evenly.
4. You add one more server → `N` changes → almost every request now goes to a different server.
5. This destroys all the useful **cached data** on the servers.
6. What we really want: when a server is added, only a **small** part of the requests should move.
7. That requirement is what **consistent hashing** is designed for.

Consistent hashing matters whenever you build **systems that need to scale to a very large size**.

---

## 1. Starting from zero: what is a "server"?

### 1.1 Imagine this

You have written a cool algorithm and it's running on your computer — just one box. Say it's a **facial recognition algorithm that adds a moustache to a person's photo**.

Someone likes it and says: *"I'll pay you money every time I use your algorithm"* (or at the end of every month — doesn't matter).

That person has a mobile phone. From the phone, they connect to your computer, send their photo, your algorithm runs, and they get back the photo with a moustache.

> The instructor says: keep money, sales, marketing etc. aside. We only care about the **technical requirement** — the algorithm must keep running properly so the customer is happy (and so you are happy).

### 1.2 Diagram (as drawn on the board)

```
   User (with phone)                         Computer
      o                  Request            +-----------+
     /|\  ---------------------------->     | algorithm |   <-- "Server"
     / \  <----------------------------     |   code    |
                         Response           +-----------+
```

### 1.3 The terms

| Term | Meaning in this example |
|---|---|
| **Server** | A computer running your program. It is called a server because it **serves requests**. |
| **Request** | What the user sends when they connect — "please run your algorithm on my photo." |
| **Response** | What the server sends back — the photo with a moustache added. |

So in one line: **a server takes requests and sends back responses.**

---

## 2. The scaling problem

### 2.1 What happens when you go viral

The first user is extremely happy. They tell all their friends. Now **thousands and thousands of requests** are coming in.

Your single computer **cannot handle this anymore**.

Since all these people are paying you, you can afford to **buy another computer — a second server**.

### 2.2 The new problem this creates

With two servers (board diagram: two server boxes, two users, arrows pointing at both servers), a new question appears:

> When a request comes in from a user, **should it go to server 1 or server 2?**

In general, you'll have **N servers**, not just two.

### 2.3 Load

Every request is work that a server must process. The total work sitting on a server is called its **load**.

You don't want one server drowning in requests while another sits idle. At the most basic level, you want the load **balanced evenly across all N servers**.

### 2.4 Load Balancing (definition)

> **Load balancing** = taking N servers and trying to spread the load **evenly** across all of them.

This is the simple problem we are solving in this lecture: take incoming requests and distribute them evenly over N servers. Consistent hashing is the concept that will help us do this *well*.

---

## 3. The simple approach: hash the request ID, then take modulo N

### 3.1 The setup: request IDs

Every request carries a **request ID**.

- Assume that when the mobile sends a request, it **randomly generates a number from `0` to `M - 1`** and uses it as the request ID.
- So the request IDs are **uniformly random** — every number in that range is equally likely.

Board: `Request ID → 0 to M - 1`

(`M` = total size of the ID space. Later, in the pie-chart example, we'll use `M = 100`.)

### 3.2 The steps

For a request with ID `r1`:

1. **Hash it:** pass `r1` through a hash function `h`. You get a number, call it `m1`.
2. **Take modulo N:** compute `m1 % N` (the remainder when dividing by N, the number of servers).
3. **Send it:** the result is a server index. Send the request to that server.

Board: `h(r1) → m1 % n` → arrow to a server.

**Why modulo N?** Because the remainder after dividing by `N` is always one of `0, 1, 2, …, N-1`. That is exactly the list of valid server indexes. So `% N` is a simple way to turn *any* number into a valid server number.

### 3.3 Worked example with 4 servers (N = 4)

Servers: `S0, S1, S2, S3`

> **How to read the board notation:** `h(10) → 3 % 4` means "the hash of request ID 10 comes out as 3; then take 3 % 4."

| Request | Request ID | Hash value `h(ID)` | `hash % 4` | Goes to |
|---|---|---|---|---|
| r1 | 10 | 3 | 3 % 4 = **3** | **S3** |
| r2 | 20 | 15 | 15 % 4 = **3** | **S3** |
| r3 | 35 | 12 | 12 % 4 = **0** | **S0** |

(The hash values 3, 15, 12 are just example outputs the instructor picked — "let's say h(20) somehow gives you 15.")

Don't worry that two out of three went to S3 — with only 3 requests that's just chance. What matters is what happens with a large number of requests.

### 3.4 Why this spreads the load evenly

- The request IDs are **uniformly random**.
- A good hash function also produces **uniformly random** outputs.
- So `hash % N` lands on each of the N servers with equal probability.

If there are **X** total requests:

- each server gets about **X / N** requests
- the **load factor** (fraction of total load on each server) is **1 / N**

Board: `X / n → 1 / n`

So far, **everything looks perfect** — this seems to be all we need.

**Except...** what happens when N changes?

---

## 4. The problem: adding one more server

### 4.1 The situation

Your customers keep loving you, you go viral, traffic keeps increasing. You need **more servers**, so you add **S4**. Now `N = 5`.

The formula is `hash % N`, and **N just changed from 4 to 5**. So every request's server must be recalculated.

### 4.2 Recalculating the same three requests (N = 5)

| Request | Hash value | Old: `% 4` | New: `% 5` | Did it move? |
|---|---|---|---|---|
| r1 | 3 | S3 | 3 % 5 = 3 → **S3** | No, still S3 ✅ |
| r2 | 15 | S3 | 15 % 5 = 0 → **S0** | **Yes** — S3 → S0 ❌ |
| r3 | 12 | S0 | 12 % 5 = 2 → **S2** | **Yes** — S0 → S2 ❌ |

The instructor's words: the requests that were being served are now **"completely bamboozled"** — thrown all over the place.

Adding just **one** server caused **2 out of 3** requests to change servers. The pie chart below shows that this isn't bad luck — it happens to a huge fraction of the whole ID space.

---

## 5. The pie chart: how much actually changes?

### 5.1 Setup

- Take the full ID space as `M = 100` numbers.
- The instructor calls these numbers **"buckets"** — *a bucket just means a number*.
- Picture it as a pie (a circle). Each server owns a slice of the pie.

### 5.2 Before: 4 servers

Each server owns 25% = 25 numbers.

```
 0          25          50          75          100
 |----S0----|----S1----|----S2----|----S3----|
    (25)        (25)        (25)        (25)
```

### 5.3 After: 5 servers

Each server should own 20% = 20 numbers. The boundaries shift to 20, 40, 60, 80.

```
 0        20        40        60        80        100
 |---S0---|---S1---|---S2---|---S3---|---S4---|
   (20)     (20)     (20)     (20)     (20)
```

### 5.4 Server-by-server walk-through (as done on the board)

- **S0:** used to own 0–24, now owns only 0–19. It **loses 5 buckets** (20–24).
- **S1:** must **take those 5 buckets** (+5). But its range now ends at 40 instead of 50, so it **loses 10 buckets** (40–49).
- **S2:** must **take those 10 buckets** (+10). Its range now ends at 60 instead of 75, so it **loses 15 buckets** (60–74).
- **S3:** must **take those 15 buckets** (+15). Its range now ends at 80 instead of 100, so it **loses 20 buckets** (80–99). It keeps only 5 of its old buckets (75–79). Total = 5 kept + 15 gained = 20 ✓
- **S4 (new):** must **take the 20 buckets** S3 lost (+20).

### 5.5 Summary table

| Server | Old range | New range | Kept | Lost | Gained |
|---|---|---|---|---|---|
| S0 | 0–24 | 0–19 | 20 | 5 | 0 |
| S1 | 25–49 | 20–39 | 15 | 10 | 5 |
| S2 | 50–74 | 40–59 | 10 | 15 | 10 |
| S3 | 75–99 | 60–79 | 5 | 20 | 15 |
| S4 | — | 80–99 | 0 | 0 | 20 |
| **Total** | | | **50** | **50** | **50** |

### 5.6 The total "cost of change" (board calculation)

The instructor adds up every change (each loss and each gain):

```
5 + 5 + 10 + 10 + 15 + 15 + 20 + 20 = 100 = M
```

**100 = M, the entire search space.** Adding just one server caused a change equal to the size of the whole ID space.

> **Clarification on the counting (not stated in the lecture, but useful to avoid confusion):**
> The sum 100 counts every moved bucket **twice** — once when one server loses it and once when another server gains it. The number of *distinct* numbers that changed owner is `5 + 10 + 15 + 20 = 50`, i.e. **half of all requests** now go to a different server. Either way, the conclusion is the same: a massive part of the system gets disturbed.
>
> Also, the pie chart assumes each server owns one continuous range. With the actual `hash % N` formula it's even worse: a hash value keeps the same server when going from `% 4` to `% 5` only if `hash % 20` is 0, 1, 2 or 3 — that's just 4 out of every 20 values. So **only ~20% of requests stay, ~80% move.** This matches the example above, where 2 of the 3 requests moved.

---

## 6. "So what if everything moved?" — Why this is a big deal

At this point you might think: *"Okay, 100 changes, everything got reassigned... so what? Requests still get served."*

It's a big deal because of the following.

### 6.1 In real systems, the request ID is NOT random

We assumed the request ID was a random number. In practice it is **never or rarely random**. The request ID usually **contains information about the user**, for example the **user ID**.

(The instructor says he'll keep calling it `r1`, but you can think of it as `u1` — a user.)

### 6.2 Same user → same hash → same server, every time

- Suppose the ID is the user name `"Gaurav"`.
- A hash function is **deterministic** ("constant"): hashing `"Gaurav"` gives the **same result every single time**.
- Same hash value → same value of `hash % N` → **Gaurav is sent to the same server again and again.**

### 6.3 Using that fact: local caching

Since Gaurav always lands on the same server, we can take advantage of it.

**Imagine this:** every time Gaurav sends a request, the server needs his **profile**, which lives in the **database (DB)**. Fetching from the database on every single request is slow and wasteful.

The smart move: after fetching it once, **store it in that server's local cache**. Next time Gaurav comes (and he'll come to this same server), just read it from the cache.

Board diagram:

```
            +--------+
            | Cache  |
            +--------+
                |
  request --> [ S0 ] ------> ( DB )
```

So the policy becomes: **route users to specific servers based on their user ID, and store relevant user data in those servers' caches.**

- 1st request from Gaurav → S0 fetches profile from DB → saves it in S0's cache
- Later requests from Gaurav → go to S0 again → profile read from cache (fast, no DB call)

### 6.4 What adding a server does to this

When N changes from 4 to 5:

- The **entire system changes**.
- Users are **haphazardly sent to different servers** than before.
- Gaurav might now land on S2 — but his profile is cached on S0. S2 has nothing, so it must go to the DB again.
- This happens for almost every user at once.
- So **all the useful cache information is basically dumped — almost all of it is now useless**, because the ranges of numbers each server serves have completely changed.

### 6.5 The key requirement

> **What we want to avoid: a huge change in the range of numbers each server is serving.**

If a server serves some range, then when a server is added, that range should change **only a tiny bit**, not completely.

---

## 7. What we actually want: the minimum-change pie chart

### 7.1 The idea (second pie chart on the board)

Start again from the 4-server pie, each server at 25%. We add S4, which needs 20% of the total.

Instead of shifting all the boundaries around the circle, **take a small slice from each of the four existing servers** and give those slices to S4:

- a little from S0
- a little from S1
- a little from S2
- a little from S3

such that **the four small slices add up to 20%** (because with 5 servers, each should hold 20%).

Board diagram: a circle split into 4 quarters, with a small triangular slice cut from each quarter toward the center — those four slices together belong to the new server.

### 7.2 Working out the numbers (M = 100)

Each old server must drop from 25 to 20, so each gives up **5 buckets** to S4.

| Server | Before | Gives to S4 | After |
|---|---|---|---|
| S0 | 25 | 5 | 20 |
| S1 | 25 | 5 | 20 |
| S2 | 25 | 5 | 20 |
| S3 | 25 | 5 | 20 |
| S4 | 0 | receives 5 × 4 = 20 | 20 |

- Load is still perfectly balanced: 20% each.
- Only **20 out of 100** numbers change owner.
- The other **80%** of users stay on their old server, so **their cached data is still valid.**

Notice that S4's share is **not one continuous range** — it's made of four small pieces scattered around the circle. That's what lets every existing server give up only a little.

### 7.3 Comparison

| Method | Numbers that change server when going 4 → 5 servers (out of 100) |
|---|---|
| `hash % N` (actual modulo) | ~80 |
| Pie with continuous ranges (board example) | 50 distinct (sum of changes = 100) |
| **Ideal: small slice from every server** | **20** |

In general terms: the ideal is to move only the share the new server needs — with 5 servers, that's just 1/5 of the requests — and leave everyone else alone.

---

## 8. Where consistent hashing comes in

- The **old, standard way of hashing** (`hash % N`) **does not work** when servers are added, because changing N reshuffles almost every request and throws away cached data.
- We need a more advanced approach that keeps load balanced **and** makes the change minimal when servers are added.
- **That approach is consistent hashing**, which is the topic that the following part of the lecture teaches.

---

## 9. Quick revision sheet

- **Server:** a computer that serves requests (takes a request, sends a response).
- **Scaling:** too many requests → add more servers → must decide where each request goes.
- **Load balancing:** spreading load evenly over N servers.
- **Simple method:** `server = hash(requestID) % N`.
  - Works because uniform IDs + uniform hash → each server gets X/N requests (load factor 1/N).
- **Flaw:** adding a server changes N → most requests map to a different server.
  - Example: hash 15 → `%4 = 3` but `%5 = 0`; hash 12 → `%4 = 0` but `%5 = 2`.
  - Pie chart with M = 100: total change = 5+5+10+10+15+15+20+20 = 100 = M (whole space).
- **Why it hurts:** request IDs are really user IDs → deterministic hash → same user always hits the same server → servers cache user data (e.g. profile from DB) → changing N sends users elsewhere → cache becomes useless.
- **Goal:** when adding a server, take a small slice from every existing server (5% each from 4 servers = 20% for the new one), so the overall change is minimal.
- **Solution:** consistent hashing.

---

## 10. Interview-style questions

**Q1. What is load balancing?**
Distributing incoming requests (load) evenly across N servers so no single server gets overloaded.

**Q2. How can you assign requests to servers using hashing?**
Hash the request ID and take the remainder with the number of servers: `server = h(id) % N`. With uniformly distributed IDs and a good hash function, each server gets about 1/N of the load.

**Q3. What is the main problem with `hash % N`?**
When the number of servers changes, N changes, so the result of `% N` changes for most keys. Nearly all requests get remapped to different servers, even though only one server was added.

**Q4. Why does remapping matter if requests still get served correctly?**
Because real request IDs contain user identity (like user ID). Since hashing is deterministic, a user consistently reaches the same server, so servers cache data for "their" users (for example, a profile fetched from the DB). A massive remap sends users to servers that don't have their cached data, so the caches become useless and the data has to be fetched from the database again.

**Q5. What property do we want from a better scheme?**
Load stays balanced, and when a server is added, only a small fraction of keys move — ideally just the share the new server needs (1/5 of keys when going to 5 servers), taken a little from each existing server. The rest of the keys keep their server. Consistent hashing provides this.

**Q6. With M = 100 and servers going from 4 to 5 using continuous ranges, what is the total change?**
5 + 5 + 10 + 10 + 15 + 15 + 20 + 20 = 100, which equals the whole search space M (50 distinct numbers change owner, each counted once as lost and once as gained).
