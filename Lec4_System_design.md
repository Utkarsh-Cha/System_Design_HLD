# System Design — Lecture 3: Consistent Hashing (Study Notes)

> **How to read these notes**
> - Everything here comes from the lecture transcript and the whiteboard.
> - The whiteboard showed servers and requests as *dots on a circle*, not as numbers. To make the dry runs concrete, I placed them at specific numbers. Those numbers are marked **(illustrative)**. The *results* (who serves what, which loads change) match what the instructor drew.
> - Section 10 (code) is clearly marked as **beyond the lecture**, because the instructor only said the code is in the video description; nothing was shown on screen.
> - Section 11 lists small slips on the board/transcript so they don't confuse you.

---

## Table of Contents

1. The real problem: adding and removing servers
2. Idea 1 — Turn the hash array into a ring
3. Idea 2 — Put the servers on the same ring
4. The routing rule: walk clockwise to the nearest server
5. Adding a server
6. Removing a server (a crash)
7. The practical problem: skewed load
8. The fix: virtual servers (multiple hash functions)
9. Where consistent hashing is used
10. Implementation sketch (beyond the lecture)
11. Slips on the board / transcript
12. Revision sheet
13. Interview questions

---

## 1. The Real Problem: Adding and Removing Servers

### 1.1 Where the lecture starts

The lecture begins in the middle of a thought ("like we saw"), continuing from the previous approach, which was:

```
server = h(requestID) % N        (N = number of servers)
```

Every request has an ID. You hash the ID, take the remainder with N, and that number tells you which server handles the request.

### 1.2 The instructor's opening point

> **The problem is not actually load balancing.**

- The `% N` approach already spreads requests evenly across servers, because a good hash function gives uniformly random outputs. So "balance the load" is already solved.
- **The actual problem is adding and removing servers.** When a server is added or removed, `N` changes. When `N` changes, `h(requestID) % N` gives a different answer for almost every request.
- Servers keep **local data** for the requests they serve (for example, cached information about the users they have been handling). If almost every request suddenly goes to a different server, that local data becomes useless everywhere. The instructor says adding/removing servers "completely changes the local data that we have in each server."

### 1.3 Small example to feel the problem (illustrative)

Suppose hash values are 0 to 19. We go from 4 servers to 5 servers.

| Hash value | `% 4` (old server) | `% 5` (new server) | Moved? |
|---|---|---|---|
| 0 | 0 | 0 | No |
| 1 | 1 | 1 | No |
| 2 | 2 | 2 | No |
| 3 | 3 | 3 | No |
| 4 | 0 | 4 | **Yes** |
| 5 | 1 | 0 | **Yes** |
| 6 | 2 | 1 | **Yes** |
| 7 | 3 | 2 | **Yes** |
| 8 | 0 | 3 | **Yes** |
| 9 | 1 | 4 | **Yes** |
| 10–19 | ... | ... | **All move** |

Only 4 out of 20 values stay put. **80% of requests move** just because we added one server.

### 1.4 What we want instead

1. Requests should still be spread evenly (keep the good part of hashing).
2. When a server is added or removed, **only a small number of requests should move**, and the rest should stay on the server they were already on.

Consistent hashing gives both.

---

## 2. Idea 1 — Turn the Hash Array Into a Ring

### 2.1 Intuition first

Imagine the numbers `0, 1, 2, ..., M-1` written along a straight ruler. Now bend the ruler into a circle so that the end (`M-1`) touches the start (`0`). It's like a clock face: after the last number, you come back to the first.

### 2.2 What stays the same

- We **still hash requests by their IDs**. Written on the board:
  ```
  Request ID  ->  h(r_i)
  ```
- The hash value, taken `% M`, gives a position between `0` and `M-1`.

### 2.3 What is different

- Before, you could picture an **array** with slots `0` to `M-1` (the board shows this array drawn above the circle).
- Now picture those same slots arranged as a **ring**: `0, 1, 2, 3, ..., M-1`, and then `M-1` "sticks to" `0`. The instructor calls it a "ring of hash."
- `M` is the **search space**: the size of the hash range. Note that we take the remainder with `M`, **not with N** (number of servers). `M` is fixed, so adding or removing a server does not change where any request lands on the ring. This is the root of why consistent hashing avoids the problem in Section 1.

### 2.4 On the whiteboard

- A large circle representing the hash ring.
- Slots labeled clockwise: `M-1, 0, 1, 2, 3, ..., M-1` (showing the wraparound).
- Shaded marks on the ring = positions where requests landed after hashing. Many requests → many marks around the ring.

### 2.5 Why a ring and not an array?

In the next step we will "walk forward" from a request to find a server. On an array, if you are near the end, there may be nothing ahead of you. On a ring, walking past `M-1` simply continues at `0`, so **there is always a next server**.

---

## 3. Idea 2 — Put the Servers on the Same Ring

### 3.1 What we do

The requests need to be sent to servers. Servers also have IDs. At first there were **four servers**, with IDs written as `0, 1, 2, 3, 4` on the board.

We **hash the server IDs too**, and take the remainder with `M`:

| Server ID | Position on ring |
|---|---|
| 0 | `h(0) % M` |
| 1 | `h(1) % M` |
| 2 | `h(2) % M` |
| 3 | `h(3) % M` |
| 4 | `h(4) % M` |

### 3.2 Same hash function or a different one?

The instructor says you can use **the same hash function as for requests, or a different one: it doesn't really matter.**

Why it doesn't matter: all we need is that servers land at spread-out, random-looking positions on the ring. Any good hash function does that.

### 3.3 Worked example (from the board)

```
M = 30
h(0) = 49
h(0) % 30 = 49 % 30 = 19
```

So this server lands at **position 19** on the ring. The instructor labels it **S1** on the ring. (Server IDs start from 0 on the board, but ring labels start from S1; it's just naming.)

The other servers are hashed the same way and land at other points: **S2, S3, S4** (drawn as colored sections on the ring).

### 3.4 Result

Now **both requests and servers are points on the same circle**.

---

## 4. The Routing Rule: Walk Clockwise to the Nearest Server

### 4.1 The rule

> When a request lands on the ring, **go clockwise** from its position and find the **nearest server**. That server serves the request.

The instructor: "That's it. Simple algorithm."

Another way to picture it: each server "owns" the stretch of ring (an **arc**) that starts just after the previous server and ends at itself. Every request landing in that stretch belongs to that server.

### 4.2 Dry run (illustrative positions, same loads as the board)

Setup:
- `M = 100` (positions 0 to 99)
- Servers: **S1 at 10, S4 at 35, S2 at 60, S3 at 85**
- Requests at: **5, 20, 50, 70, 95**

```
clockwise ──►

 pos:  0 .. 5 .. 10 ...... 20 ...... 35 ...... 50 ...... 60 ...... 70 ...... 85 ...... 95 .. 99
            r   [S1]       r        [S4]       r        [S2]       r        [S3]       r
       ▲                                                                                      │
       └────────────────────────── wraps: after 99 comes back to 0 ───────────────────────────┘

 r = request, [S] = server
```

| Request at | Walk clockwise... | First server hit | Served by |
|---|---|---|---|
| 5 | 6, 7, 8, 9, 10 | S1 at 10 | **S1** |
| 20 | 21 ... 35 | S4 at 35 | **S4** |
| 50 | 51 ... 60 | S2 at 60 | **S2** |
| 70 | 71 ... 85 | S3 at 85 | **S3** |
| 95 | 96 ... 99, **wrap to 0**, 1 ... 10 | S1 at 10 | **S1** |

The last row is the **wraparound case**. The request is past the last server, so it goes around the ring and ends up at S1. The instructor points this out on the board: "in fact, it goes over here to S1."

**Loads (matching the board):**

| Server | Load |
|---|---|
| S1 | 2 |
| S2 | 1 |
| S3 | 1 |
| S4 | 1 |

### 4.3 Why this design gives balanced load: expected load = 1/N

The board has a boxed value: **`1/N`**. Here is the reasoning chain:

1. **Hash outputs are uniformly random**, so server positions are scattered randomly around the ring.
2. Because the positions are uniformly random, you can expect the **distances between servers to be uniform too**. On average, each server's arc covers about `1/N` of the ring.
3. **Requests are also uniformly spread**, so the number of requests in an arc is proportional to how long the arc is.
4. Distance uniform → load uniform. So each server's **expected load factor is `1/N`** of all requests.

> **Instructor's note:** "That's the smart bit, but you already had that earlier." The `% N` method also gave `1/N`. **Balanced load is NOT what makes consistent hashing special.** What makes it special is what happens when servers are added or removed (next two sections).

---

## 5. Adding a Server

### 5.1 What happens on the board

- A fifth server is added: `h(5) % M` is written on the right side.
- It lands on a point that sits inside the arc that used to belong to **S3**.
- (The instructor calls this new server "S4" on the board. That's a slip, because S4 already exists. These notes call it **S5**.)

### 5.2 The logic

- The new server sits between S3 and the server before S3.
- Requests that land **before** the new server's position (in that arc) now reach the new server **first** when walking clockwise → they move to the new server.
- Requests that land **after** the new server's position still walk on to S3 → they stay on S3.
- For every other request on the ring, the clockwise walk never passes the new point → **their server does not change**.

Board numbers: S3's load was **3**. After adding the new server, two of those requests find the new server as their nearest clockwise server:

| Server | Before | After |
|---|---|---|
| S3 | 3 | **1** |
| New server (S5) | 0 | **2** |
| S1, S2, S4 | unchanged | unchanged |

> "**Only S3 is affected.** S1 is not affected, S4 is not affected, S2 is not affected." The change in each server's load is much smaller than with the `% N` method.

### 5.3 Dry run (illustrative, continuing from Section 4)

To match the board's "S3 had 3 requests," suppose two more requests arrived at **65** and **68**. Now S3 serves 65, 68, 70.

New server **S5 lands at 69**.

```
BEFORE:  [S2]@60 ---- r@65 -- r@68 ---------- r@70 ---------- [S3]@85   → S3 serves 65, 68, 70
AFTER:   [S2]@60 ---- r@65 -- r@68 -- [S5]@69 -- r@70 -------- [S3]@85   → S5 serves 65, 68;  S3 serves 70
```

| Request | Before | After | Moved? |
|---|---|---|---|
| 5 | S1 | S1 | No |
| 20 | S4 | S4 | No |
| 50 | S2 | S2 | No |
| 65 | S3 | **S5** | Yes |
| 68 | S3 | **S5** | Yes |
| 70 | S3 | S3 | No |
| 95 | S1 | S1 | No |

Loads now: **S1 = 2, S4 = 1, S2 = 1, S5 = 2, S3 = 1** (total 7).

Only the requests that the new server should take moved, and all of them came from **one** server. Compare with Section 1.3, where adding a server moved 80% of requests.

---

## 6. Removing a Server (A Crash)

### 6.1 What happens on the board

- **S1 goes down.** For some reason there was a crash; the instructor jokes it "lost its power cord." S1 is crossed out on the board.
- The requests S1 was serving now continue walking clockwise past S1's old position and reach the **next server clockwise**, which on the board is **S4**. The arrows are redrawn to S4.
- "S4 is a happy guy." ... "Not really." (See Section 7.)

### 6.2 Why only S1's requests move

Every other request reaches its own server **before** it would ever reach S1's old position. Nothing on their clockwise path changed, so their server doesn't change. Only requests whose walk used to stop at S1 now keep walking.

### 6.3 Dry run (illustrative, continuing from Section 5)

State: requests at 5, 20, 50, 65, 68, 70, 95. Servers: S1@10, S4@35, S2@60, S5@69, S3@85.

**Remove S1.**

```
BEFORE:  r@95 --wrap--> r@5 -- [S1]@10 -- r@20 -- [S4]@35    → S1 serves 95, 5;  S4 serves 20
AFTER:   r@95 --wrap--> r@5 --  (S1 ✗)  -- r@20 -- [S4]@35    → S4 serves 95, 5, 20
```

| Server | Load before | Load after |
|---|---|---|
| S1 | 2 | — (removed) |
| S4 | 1 | **3** |
| S2 | 1 | 1 |
| S5 | 2 | 2 |
| S3 | 1 | 1 |

- 7 requests, 4 servers → the fair share would be 7/4 = 1.75 each.
- S4 now has 3 of 7 ≈ **43%**. That is the instructor's point: **about half the load is on a single server.**

---

## 7. The Practical Problem: Skewed Load

### 7.1 Theory vs practice

- **Theory:** expected load per server = `1/N`.
- **Practice:** you can get **skewed distributions**, meaning some servers get far more than their share.

### 7.2 Why skew happens

- **You do not have enough servers**, so there aren't enough points on the ring.
- With only a few random points, they can easily bunch up together. Then some arcs are huge and some are tiny, so some servers get huge loads and others get tiny ones.
- The instructor: if you had a lot of servers, "you would have a lot of red points and it would be evenly distributed," and the chance of skew would be really low. But with just four, you can end up with **about half the load on one server, which is "terrible."**
- Removal makes it worse: when a server dies, its **entire** arc goes to **exactly one** neighbor (the next server clockwise), so that one neighbor's load can double or triple, as S4's did.

### 7.3 Where we stand (the instructor's recap)

| We know... | Status |
|---|---|
| How to add servers and map requests to them | ✅ |
| How to remove servers and reassign their requests | ✅ |
| That theoretically this is the **minimum change** | ✅ |
| How to make it work **in practice** (no skew) | ❓ |

> "This is the place where system design engineers actually solve problems." The instructor pauses here and asks you to try to think of a solution yourself.

**Hint to the solution:** the problem is really *too few points on the ring*, not *too few machines*. So can we get more points without buying more machines?

---

## 8. The Fix: Virtual Servers (Multiple Hash Functions)

### 8.1 Intuition first

Imagine each server gets several **name tags**, and each name tag is stuck at a different spot around the ring. It's still one physical machine, but it shows up in several places. Walking clockwise, a request might hit any of a server's name tags; whichever tag it hits, that server serves the request.

### 8.2 What "virtual server" does NOT mean

- It does **not** mean virtual machines (virtual boxes).
- It does **not** mean buying more servers, because those are expensive.

### 8.3 What it DOES mean: k hash functions

- Instead of one hash function `h` for the servers, use several: **`h1, h2, ..., hk`**.
- Pass **each server ID through each of the k hash functions** → each server gets **k different positions** on the ring.
- Take each result `% M` as before.

On the board, `h_1, h_2, ..., h_k` is written next to the server hash list, with **"k hash functions"** written above it. S3 is shown mapping to two more points around the ring, and S4 also to two more points.

**Counting points:**

```
Total points on ring = k × N
Example: k = 3, N = 4  →  12 points instead of 4
```

**Routing rule is unchanged:** a request walks clockwise to the nearest point, and the server that owns that point serves it.

### 8.4 How to choose k

Written on the board above "k hash functions": **`log(N)` or `log(M)`**.

If you choose `k` appropriately (like `log N` or `log M`), you can **almost entirely remove the chance of skewed load** on one server.

### 8.5 Why more points fixes skew

- A server's total load is now the **sum of k separate arcs** scattered around the ring.
- Some of those arcs will be big and some small, but added together they **average out**.
- One unlucky big gap no longer decides a server's entire load.
- More points on the ring overall = the "lots of red points, evenly distributed" situation the instructor described, without buying machines.

### 8.6 Removing a server with virtual servers

- Remove **all k points** of that server.
- Each removed point's requests walk clockwise to **that point's** nearest neighbor.
- Because the removed points are scattered around the ring, their neighbors are generally **different servers**.
- So the dead server's load gets split across many servers. The instructor: you "take some load from this server, some load from this server, some load from this server..." and every server's load increases **expectedly uniformly**.
- Board: a **pie chart** in the top-right corner showing the load spread evenly across all servers.

### 8.7 Adding a server with virtual servers

The same thing happens in reverse: the new server adds k points, and each point takes a small slice from whichever server was next clockwise. The new server takes a little from many servers. You get **expected minimum change** in the number of requests each server handles.

### 8.8 Dry run (illustrative): 4 servers, k = 3, M = 100

12 points placed around the ring. "Arc length" = how many positions a point owns (from just after the previous point up to itself).

| Point position | Owner | Arc it owns | Arc length |
|---|---|---|---|
| 5 | S1 | 95–99, wrap, 0–5 | 11 |
| 12 | S3 | 6–12 | 7 |
| 20 | S2 | 13–20 | 8 |
| 28 | S4 | 21–28 | 8 |
| 37 | S1 | 29–37 | 9 |
| 45 | S2 | 38–45 | 8 |
| 53 | S3 | 46–53 | 8 |
| 61 | S4 | 54–61 | 8 |
| 70 | S2 | 62–70 | 9 |
| 78 | S1 | 71–78 | 8 |
| 86 | S4 | 79–86 | 8 |
| 94 | S3 | 87–94 | 8 |

**Share of the ring per server** (fair share = 25):

| Server | Arcs added up | Share |
|---|---|---|
| S1 | 11 + 9 + 8 | 28 |
| S2 | 8 + 8 + 9 | 25 |
| S3 | 7 + 8 + 8 | 23 |
| S4 | 8 + 8 + 8 | 24 |

All close to 25. The small differences average out.

**Now remove S1** (its points at 5, 37, 78 disappear):

| Removed point | Its arc | Next point clockwise | Arc goes to |
|---|---|---|---|
| 5 | 11 | 12 | **S3** |
| 37 | 9 | 45 | **S2** |
| 78 | 8 | 86 | **S4** |

| Server | Before | After |
|---|---|---|
| S2 | 25 | 34 |
| S3 | 23 | 34 |
| S4 | 24 | 32 |

Fair share with 3 servers = 33.3, and everyone is close to it. S1's load was split across **three different servers** instead of dumped on one. Compare with Section 6, where one neighbor got everything.

### 8.9 Side-by-side comparison

| Situation | Without virtual servers | With virtual servers (k points each) |
|---|---|---|
| Points on ring | N | k × N |
| Load in practice | Can be badly skewed with few servers | Close to 1/N; skew almost eliminated with good k |
| Server removed | Whole load goes to **one** neighbor | Load split across **many** servers |
| Server added | Takes load from **one** neighbor | Takes a little from **many** servers |
| Cost | — | No extra machines; just more hash computations |

---

## 9. Where Consistent Hashing Is Used

- **Load balancing** is used extensively in **distributed systems**, and consistent hashing is a key way to do it.
- Used by **web caches**.
- Used by **databases**.
- What it gives you: **flexibility** (add/remove servers easily) plus **load balancing**, in a clear and efficient way.
- The instructor says you should definitely know this concept. Code was shared in the video description (not shown on screen).

---

## 10. Implementation Sketch (Beyond the Lecture)

> Not shown in the lecture. Included so you can see the lecture's algorithm as code. It follows exactly the steps described above.

```python
import bisect
import hashlib

class ConsistentHashRing:
    def __init__(self, k=3, M=2**32):
        self.k = k                 # virtual points per server
        self.M = M                 # size of the ring (search space)
        self.positions = []        # sorted list of all point positions on the ring
        self.owner = {}            # position -> server ID

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % self.M

    def add_server(self, server_id):
        for i in range(self.k):                       # k "hash functions"
            pos = self._hash(f"{server_id}#{i}")
            bisect.insort(self.positions, pos)        # keep ring points sorted
            self.owner[pos] = server_id

    def remove_server(self, server_id):
        for i in range(self.k):                       # remove all k points
            pos = self._hash(f"{server_id}#{i}")
            self.positions.remove(pos)
            del self.owner[pos]

    def get_server(self, request_id):
        pos = self._hash(str(request_id))             # request lands on ring
        idx = bisect.bisect_left(self.positions, pos) # first point at or after pos = "clockwise nearest"
        if idx == len(self.positions):                # walked past M-1 → wrap to 0
            idx = 0
        return self.owner[self.positions[idx]]
```

### 10.1 How the code maps to the lecture

| Lecture idea | Code |
|---|---|
| Ring of positions 0..M-1 | `% self.M`, and `positions` sorted in increasing order (= clockwise) |
| Hash server IDs onto the ring | `add_server` |
| k hash functions h1..hk | `self._hash(f"{server_id}#{i}")` for `i = 0..k-1`. Adding a different suffix to the key makes each one behave like a different hash function. |
| Walk clockwise to nearest server | `bisect_left` finds the first server point at or after the request's position |
| Wraparound (M-1 sticks to 0) | `if idx == len(self.positions): idx = 0` |
| Remove server = remove its k points | `remove_server` |

Note: this sketch ignores the rare case where two points hash to the exact same position.

### 10.2 Complexity (with P = k × N points)

| Operation | Time |
|---|---|
| `get_server` (route a request) | O(log P), binary search |
| `add_server` / `remove_server` | O(k · P) with a Python list (inserting/removing shifts elements); O(k log P) with a balanced tree |
| Memory | O(P) |

### 10.3 Quick experiment confirming the lecture's claim

I ran this code with 4 servers and 20,000 requests:

| k | Requests per server | Observation |
|---|---|---|
| 1 | 17762 / 1755 / 437 / 46 | One server got ~89% of traffic, the exact skew problem from Section 7 |
| 100 | 6015 / 4897 / 4629 / 4459 | Close to the fair 5000 each |

When one server was removed with k = 100, only that server's requests moved (6015 of 20,000), and they spread across all three remaining servers.

---

## 11. Slips on the Board / Transcript

These don't change the concept, but they can confuse you if you watch the video:

1. **"If I hash 0 mod M is 30"** means **M = 30**; then `h(0) = 49` and `49 % 30 = 19`.
2. **Server ID 0 is called S1** on the ring. Server IDs on the board start at 0; ring labels start at S1.
3. **The new fifth server is called "S4"** even though S4 already exists. It's a new server; these notes call it S5. The on-screen description calls it a "virtual node," but virtual servers haven't been introduced at that point, so it's just the newly added server.
4. **S3's load changes from 1 to 3 between drawings.** In the first drawing S3 had 1 request; in the add-server example it had 3. More requests had been drawn on the ring by then. What matters is the mechanism: 2 of S3's 3 requests moved to the new server.
5. **"log N or log M"** on the board is the suggested value of **k** (number of hash functions), not a hash output.

---

## 12. Revision Sheet

| Concept | One-line explanation |
|---|---|
| Real problem | Not load balancing; it's that adding/removing servers with `% N` remaps almost every request and wipes out servers' local data |
| Hash ring | Positions `0..M-1` arranged in a circle; `M-1` wraps to `0` |
| Requests on ring | `h(requestID) % M` |
| Servers on ring | `h(serverID) % M` (same or different hash function, doesn't matter) |
| Routing rule | Walk clockwise to the nearest server |
| Expected load | `1/N`, because uniform hashes → uniform gaps → uniform load |
| Add server | Only the one server just clockwise of it loses some requests |
| Remove server | Its requests go to the next server clockwise; nobody else changes |
| Theoretical guarantee | Minimum change in mapping when servers change |
| Practical problem | Few servers → uneven gaps → skewed load (can be ~half on one server) |
| Fix | Virtual servers: k hash functions → k points per server → k × N points |
| Choosing k | `log N` or `log M` |
| Virtual server ≠ | A VM or an extra machine (those are expensive) |
| Remove with virtual servers | k points removed; load spreads across many servers (pie chart) |
| Used in | Distributed systems, web caches, databases |

---

## 13. Interview Questions

**Q1. Why is `h(requestID) % N` a bad way to assign requests when servers change?**
Load is balanced fine, but changing N changes the result of `% N` for almost every request. Nearly all requests move to different servers, so local data on each server (like caches) becomes useless.

**Q2. How does consistent hashing assign a request to a server?**
Hash the request ID and the server IDs onto the same ring of size M (`% M`). From the request's position, walk clockwise; the first server you reach serves it.

**Q3. Why use a ring instead of an array?**
So the clockwise walk always finds a server. Walking past `M-1` wraps around to `0`.

**Q4. Must the server hash function be the same as the request hash function?**
No. The instructor says it doesn't really matter; you only need servers spread randomly on the ring.

**Q5. Why is the expected load per server 1/N?**
Hashes are uniformly random, so gaps between servers are expected to be uniform. Requests are uniform too, so each server's share of requests is expected to be 1/N.

**Q6. What happens when a server is added?**
It lands inside one existing server's arc and takes only the requests between the previous server and itself. All other servers are unaffected.

**Q7. What happens when a server crashes?**
Its requests continue clockwise to the next server. No other server's assignments change.

**Q8. What is the practical weakness of basic consistent hashing?**
With few servers, the random positions can be uneven, so load gets skewed (for example, about half on one server). On removal, one neighbor absorbs the entire load of the removed server.

**Q9. What are virtual servers? Are they virtual machines?**
No. They are extra positions on the ring for the same physical server, created by passing the server ID through k different hash functions. N servers → k × N points.

**Q10. How do you choose k?**
Something like `log N` or `log M`. With a good k, the chance of skew is almost eliminated.

**Q11. With virtual servers, where does a removed server's load go?**
Each of its k points hands its arc to its own clockwise neighbor, and those neighbors are usually different servers. The load spreads roughly uniformly across many servers instead of landing on one.

**Q12. Where is consistent hashing used?**
Load balancing in distributed systems, web caches, and databases.
