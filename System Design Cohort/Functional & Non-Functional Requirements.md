# Functional & Non-Functional Requirements

When we design any system, the first step is to understand the requirements. They are split into two types:

1. **Functional Requirements** – *what* the system should do.
2. **Non-Functional Requirements** – *how well* the system should do it.

---

## 1. Functional Requirements

Functional requirements are the **features** of the system — the actions a user can perform. They are usually written as *"Users should be able to …"*.

If a functional requirement is missing, the product is incomplete. Each functional requirement usually becomes one API in the design.

### Example: Uber Wallet

- Users should be able to **load money** into the wallet from a **Credit Card (CC)** or **Debit Card (DC)**.
- Users should be able to **see the current balance**.
- Users should be able to **see the transaction (txn) history**.
- Users should be able to **pay another user** using the wallet.

Each of these looks simple, but has something to think about:

| Feature | API (example) | What to be careful about |
|---|---|---|
| Load money | `POST /wallet/topup` | If the request is retried, the card must not be charged twice (idempotency). |
| See balance | `GET /wallet/balance` | Balance must always be correct; a wrong balance allows spending money twice. |
| Txn history | `GET /wallet/transactions` | History keeps growing, so results are returned page by page. |
| Pay another user | `POST /wallet/transfer` | Money must leave A's wallet **and** reach B's wallet together, or neither should happen. |

---

## 2. Non-Functional Requirements

Non-functional requirements describe the **qualities** of the system. A system can have all the features and still be bad if it is slow, often down, or hard to change.

The main ones covered:

- **Scalability**
- **Availability**
- **Maintainability**
- **Fault tolerance**
- **Reliability** (homework)
- etc. (consistency, latency, security, durability…)

For the Uber Wallet: money must never be lost or counted twice, so **correctness and durability** matter more than anything. It is better to show "please try again" than to process a payment wrongly.

---

## 3. Availability

### Definition

> **Availability** is the percentage of time for which a service or feature of the whole infrastructure is **usable by the clients** and can be operated in normal conditions.

It is also called **uptime**.

### Formula

```
Availability (%) = 100 × uptime / (uptime + downtime)
```

Example: a service was up for 99 hours and down for 1 hour → 100 × 99 / 100 = **99%**.

### The "Nines"

Availability is usually described in "nines". 99% = **2 nines**, 99.9% = **3 nines**, and so on.

| Availability | Downtime per Year | per Month | per Week | per Day |
|---|---|---|---|---|
| 99% (2 nines) | 3.65 days | 7.2 hours | 1.68 hours | 14.4 minutes |
| 99.9% (3 nines) | 8.77 hours | 43.2 minutes | 10.1 minutes | 1.44 minutes |
| 99.99% (4 nines) | 52.56 minutes | 4.32 minutes | 1.01 minutes | 8.64 seconds |
| 99.999% (5 nines) | 5.26 minutes | 25.9 seconds | 6.05 seconds | 864 milliseconds |
| 99.9999% (6 nines) | 31.5 seconds | 2.59 seconds | 605 milliseconds | 86.4 milliseconds |

**How to calculate it yourself:** downtime = (100% − availability) × time period.
For 99.9% in a day: 0.001 × 24 × 60 × 60 = 86.4 seconds ≈ 1.44 minutes.

Things to notice:
- Every extra nine means **10 times less downtime**, and it is much harder and costlier to achieve.
- At 5 nines you get about 5 minutes of downtime in a whole year. No human can even notice and fix a problem that fast, so recovery has to be **automatic**.

### Why one server is never enough

If you run your service on one server and it crashes, your service is down. That server is a **Single Point of Failure (SPOF)** — one component whose failure brings down the whole system.

To get high availability we run **multiple copies** (redundancy), so if one fails, another keeps serving.

### Active–Active vs Active–Passive

**Active–Active** (shown in the lecture): two setups (e.g., two data centres, each with servers S1 and S2) both serve traffic at the same time. If one whole data centre goes down, the other one keeps handling users.

- Failover is almost instant — the traffic simply goes to the setup that is still alive.
- All machines are being used, so no capacity is wasted.
- Each side must be able to handle the full load alone if the other side fails.

**Active–Passive:** only the primary serves traffic; a standby copy waits. When the primary fails, the standby is promoted.

- Simpler (only one place is writing data).
- Failover takes some time (detect failure → switch).
- The standby machines sit idle most of the time.

### Availability depends on every component

If a request has to pass through a load balancer, an app server and a database, **all three** must be working:

```
Total = A(LB) × A(App) × A(DB)
      = 0.9999 × 0.999 × 0.999 ≈ 99.79%
```

The total is lower than any single component. Every extra dependency reduces availability.

But if you have **two app servers** and only one needs to work:

```
Total = 1 − (1 − 0.99) × (1 − 0.99) = 99.99%
```

Two servers at 99% each give 99.99% together. This is why **redundancy** improves availability.

---

## 4. Case Study: The 2017 Amazon S3 Outage

Link: https://www.gremlin.com/blog/the-2017-amazon-s-3-outage
AWS's official summary: https://aws.amazon.com/message/41926/

This is a real example of what happens when availability fails at a huge scale.

### What happened

- **Date:** 28 February 2017.
- **Where:** Amazon S3 in the **US-EAST-1** region (North Virginia).
- An AWS engineer was debugging a slow **S3 billing system**. He ran a command (from an approved playbook) to remove **a few servers**.
- **One input was typed wrong**, and a **much larger set of servers was removed**.
- Those servers supported two critical S3 subsystems:
  - **Index subsystem** – keeps the metadata and location of every object in S3. Needed for every GET, LIST, PUT and DELETE.
  - **Placement subsystem** – decides where new objects are stored. It depends on the index subsystem.
- Both subsystems had to be **fully restarted**, and S3 could not serve requests during that time.

### Timeline (Pacific time)

| Time | Event |
|---|---|
| 9:37 AM | Wrong command removes the servers; S3 goes down |
| 11:37 AM | AWS can update its status page again |
| 12:26 PM | Index subsystem can serve GET, LIST and DELETE again |
| 1:18 PM | Index subsystem fully recovered |
| 1:54 PM | Placement subsystem recovered; S3 back to normal |

Total outage: about **4 hours**.

### Why it took so long

- AWS had **not fully restarted these subsystems in years**. S3 had grown hugely in that time, so the restart (and the safety checks on all the metadata) took much longer than anyone expected.
- Placement could only recover **after** index recovered.
- Other services had a backlog of work and needed extra time to catch up.

### Who was affected

- Other AWS services that depended on S3: **EC2** (launching new instances), **EBS**, **Lambda**, the S3 console.
- Thousands of companies: Slack, Trello, GitHub, GitLab, Docker Hub, Quora, Medium, Twitch, Expedia, Square, Kickstarter, Coursera, Heroku, Business Insider, and many more.
- Even **AWS's own status dashboard** went down, because it was stored on S3 in the same region. AWS had to post updates on Twitter.

### What AWS changed after this

1. The tool now removes capacity **slowly**.
2. The tool **refuses** to remove servers if that would take a subsystem below its minimum required capacity.
3. Other operational tools were checked for the same problem.
4. Work started to make the subsystems **recover faster**.
5. The index subsystem was split into **smaller independent parts ("cells")**, so one problem can't affect everything.
6. The status dashboard now runs across **multiple regions**.

### What we learn from it

- **A single region is a single point of failure.** Companies that had copies in another region recovered quickly.
- **Humans make mistakes.** Tools should have limits so one typo can't remove too much.
- **Your monitoring/status page must not depend on the thing it is monitoring.**
- **Practice recovery.** If you never restart a system, you don't know how long it takes.
- **Know your dependencies.** EC2 and Lambda went down because they depended on S3.
- **Test failures on purpose** (chaos engineering / "GameDays") to find weak points before a real outage does.

---

## 5. Scalability

### Definition

> **Scalability** is the ability of your system to handle an **increasing amount of workload** without hampering the performance.

"Workload" can mean more users, more requests per second, more data, or heavier requests.

A scalable system stays fast as the load grows, as long as we add resources.

### How to scale a system

There are two ways:

#### Vertical Scaling (Scaling Up)

Make the **same machine bigger** — more CPU, more RAM, more disk.

- ✅ Easy — no code changes.
- ✅ No problems of coordinating multiple machines.
- ❌ There is a **limit** — you can't buy an infinitely big machine.
- ❌ It's still **one machine**, so it's still a single point of failure.
- ❌ Very big machines are very expensive.
- ❌ Upgrading usually needs a restart (downtime).

#### Horizontal Scaling (Scaling Out)

Add **more machines** and split the work between them.

- ✅ Almost no limit — just keep adding machines.
- ✅ If one machine dies, the others keep working (better availability).
- ✅ You can add or remove machines as traffic changes.
- ❌ You need a **load balancer** to distribute requests.
- ❌ Servers should be **stateless** — user data like sessions must be kept in a shared place (database/cache), not in one server's memory.
- ❌ More moving parts to manage.

### Load Balancer

When we scale horizontally, a **load balancer** sits in front of the servers and decides which server handles each request. **Nginx** is a popular load balancer.

What it does:
- Spreads requests across servers.
- Checks server health and stops sending traffic to dead servers.

Common ways to pick a server:
- **Round Robin** – one by one, in turn.
- **Weighted Round Robin** – bigger servers get more requests.
- **Least Connections** – send to the server that is least busy.
- **IP Hash** – the same user always goes to the same server.

**Important:** if you have only **one** load balancer, it becomes a **Single Point of Failure (SPF)** — if it goes down, all servers behind it are unreachable. The fix is to run more than one load balancer (with automatic failover between them), or use a managed cloud load balancer which is already replicated.

### Auto Scaling

**Auto scaling** means adding or removing servers **automatically** based on load (for example, add a server when CPU goes above 70%).

This works well when traffic grows slowly. It does **not** work well for sudden huge spikes, because a new server takes minutes to start and become ready — by then the spike has already hit. (See the Hotstar case study below.)

### Load Testing

You only know if your system scales by **testing it under load**.

- **Load test** – run at the expected peak traffic and check it's still fast.
- **Stress test** – push beyond the peak to find the **breaking point**.
- **Spike test** – a sudden jump in traffic.
- **Soak test** – normal load for many hours, to find slow problems like memory leaks.

Things to measure: requests per second, response time (especially the slowest 1% of requests — "p99"), and error rate.

Common tools: JMeter, Gatling, k6, Locust.

---

## 6. Case Study: Scaling Hotstar for 25 Million Concurrent Viewers

Link: https://www.youtube.com/watch?v=QjvyiyH4rr0
Talk by Gaurav Kamboj (Cloud Architect, Hotstar) at AWS re:Invent 2019.

*(Details below are from written summaries of the talk.)*

### The scale

- In July 2019, during the **India vs New Zealand World Cup semi-final**, Hotstar had **25.3 million people watching at the same time** — a world record. (Earlier record: 18.6 million at the IPL 2019 final.)
- Traffic grew by **more than 1 million users per minute**.
- More than **1 million requests per second**.
- More than **10 Tbps** of bandwidth — reported to be around 70% of India's internet bandwidth.

### What makes their traffic special

- Traffic depends on **what happens in the match**, not on time of day. When a famous batsman comes in, millions join within minutes. When he gets out, millions leave and go back to the home page at once — a **"tsunami"** of requests.
- **Push notifications** sent to millions of phones cause sudden spikes too.

### Why normal auto scaling failed

- A new server took a few minutes to start, but they had only about **90 seconds** to react to a spike.
- The cloud sometimes returned **"insufficient capacity"** — not enough machines of that type available.
- Auto scaling groups use a single instance type and add servers in fixed steps, which is too slow and inflexible.
- Cloud APIs **throttle** when you request too much at once.

### What they did instead

- **Pre-warming:** before a big match, they scaled up to the expected peak **in advance**, based on past data.
- **Scaling on number of viewers** (concurrency) instead of CPU, using their own custom scaling script.
- **Using many instance types** so they wouldn't run out of one type.
- **Ready-made server images** with everything pre-installed, so new servers became healthy faster.
- **Caching and CDN** so most requests never reach the main servers.
- **Client-side protection:** apps wait a random small time before retrying (jitter) and wait longer after each failure (exponential backoff), so millions of phones don't retry at the exact same moment.
- **Rate limiting** to block unwanted traffic.

### Load testing at huge scale — "Project HULK"

- Their in-house load testing setup used about **108,000 CPUs**, **200 Gbps** of network, across **8 regions**.
- They tested for **50 million** users — about **2× their expected peak**.
- They also ran **"tsunami" tests** (sudden surges).

### Chaos engineering and GameDays

- **GameDays:** they simulated a full match before the real one.
- They deliberately caused problems: push notification surges, delayed scaling, slow network, CDN failure — to find each system's breaking point.

### Panic mode (graceful degradation)

During a live match there's no time to debug. So when the system is overloaded:
- **Non-important features are switched off** — recommendations, personalisation.
- The freed-up capacity goes to **important features** — video playback, login, payments.

The user still gets to watch the match, just without extras. This is called **graceful degradation**.

### What we learn from it

- Auto scaling alone can't handle sudden spikes — **scale up in advance**.
- **Test at 2× your expected peak**, so you know your limits.
- Know which features are **critical** and protect them first.
- **Turning off features is better than going down.**

---

## 7. Maintainability

### Definition

> **Maintainability** refers to making the system as **evolvable** as possible **without major disruptions**.

In simple words: how easy it is to change, fix and run the system over time. Most of a system's life is spent *after* launch — fixing bugs, adding features, handling incidents — so this matters a lot.

The lecture covered three areas:

### 1. How to structure the code

- Split code into clear layers (e.g., controller → service → database layer), each with one job.
- Follow consistent naming and folder structure.
- Write tests so you can change code without fear of breaking things.
- Keep the design as simple as possible.
- Document important decisions.

### 2. How to monitor the application

You can't fix what you can't see. Monitoring means collecting:

- **Metrics** – numbers over time: requests per second, response time, error rate, CPU, memory.
- **Logs** – records of what happened, with enough detail to debug.
- **Traces** – the path of one request across all services.
- **Alerts** – notify the team when something goes wrong.

Good monitoring means problems are found and fixed faster, which also improves availability.

### 3. How to achieve modularity in the architecture

- Break the system into **independent modules/services** that talk through clear APIs.
- Changing one module should not require changing the others (**loose coupling**).
- Each part can then be developed, deployed and scaled on its own.
- Use small, frequent, reversible releases (CI/CD) so changes are low risk.

---

## 8. Fault Tolerance

### Definition

**Fault tolerance** is the ability of a system to **keep working even when some of its parts fail**.

- A **fault** is one part going wrong (a disk dies, a server crashes, a network link breaks).
- A **failure** is when the whole system stops working for users.
- Fault tolerance is about making sure **faults don't become failures**.

### Techniques covered

#### Replication
Keep **multiple copies** of servers and data. If one copy is lost, another one takes over. Place copies in different data centres/regions so one disaster can't destroy all of them.

#### Load balancing
The load balancer checks the health of servers and **stops sending requests to a broken server**. Users don't notice that a server died.

#### Backup systems
- **Backups** protect against data being deleted or corrupted. (Replication alone won't help here — a mistake gets copied to all replicas too.)
- **Standby systems** take over when the main one fails.
- Always **test that backups can actually be restored**.

#### Monitoring
Detect problems **before users notice**, and automatically restart or replace broken parts.

### Other useful patterns

- **Timeouts** – don't wait forever for a slow service.
- **Retries with backoff** – retry failed requests, but wait longer each time.
- **Circuit breaker** – if a service keeps failing, stop calling it for a while.
- **Graceful degradation** – switch off non-important features (like Hotstar's panic mode).
- **Limit the blast radius** – split the system into independent parts so one failure can't take down everything (like AWS splitting S3's index into cells).

---

## 9. Homework

### HW 1: Reliability

**Reliability** means the system **keeps doing the correct thing** consistently over time, even when things go wrong.

**Reliability vs Availability:**
- **Availability** asks: *Is the system up right now?*
- **Reliability** asks: *Does it keep working correctly without failing?*

Examples:
- A clock that never stops but always shows the wrong time is **available but not reliable**.
- A server that crashes for 1 second every hour is up 99.97% of the time (**highly available**), but fails 24 times a day (**not very reliable**).

Two common measures:
- **MTBF (Mean Time Between Failures)** – how long the system runs on average before it fails. Higher is better.
- **MTTR (Mean Time To Repair)** – how long it takes on average to fix it. Lower is better.

They connect to availability:

```
Availability = MTBF / (MTBF + MTTR)
```

So you can improve availability by **failing less often** or by **recovering faster**. (The S3 outage was long mainly because recovery was slow.)

How to make a system reliable:
- Remove single points of failure.
- Test properly (including load tests and failure tests).
- Release changes carefully, with an easy way to roll back.
- Put safety limits on admin tools (the S3 lesson).
- Monitor and alert.

### HW 2: Set up Nginx as a load balancer on an EC2 machine

**Goal:** one EC2 machine running Nginx as a load balancer, in front of two EC2 machines running a simple web server. Requests should be shared between the two, and the site should keep working if one of them is stopped.

**Steps:**

1. **Launch 3 EC2 instances** (free-tier size is fine).
   - Load balancer: allow port 80 from anywhere, and SSH (port 22) only from your IP.
   - Backends: allow port 80 only from the load balancer.

2. **On each backend**, install Nginx and give each one a different page so you can tell them apart:

```bash
sudo dnf install -y nginx          # Amazon Linux (Ubuntu: sudo apt install -y nginx)
echo "Hello from $(hostname)" | sudo tee /usr/share/nginx/html/index.html
sudo systemctl enable --now nginx
```

3. **On the load balancer**, install Nginx and create `/etc/nginx/conf.d/lb.conf`:

```nginx
upstream backend_servers {
    server <BACKEND_1_PRIVATE_IP>:80;
    server <BACKEND_2_PRIVATE_IP>:80;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_servers;
    }
}
```

Then check and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

(If the default Nginx config also uses port 80, remove or comment out its `server` block.)

4. **Test it:** open `http://<LB_PUBLIC_IP>` and refresh a few times, or run:

```bash
for i in 1 2 3 4 5 6; do curl -s http://<LB_PUBLIC_IP>; done
```

The responses should alternate between the two backends (round robin).

5. **Test fault tolerance:** stop Nginx on one backend (`sudo systemctl stop nginx`) and test again. All requests should now go to the other backend.

6. **Try other strategies:** add `least_conn;` or `ip_hash;` as the first line inside `upstream` and see how behaviour changes.

7. **Stop or terminate the instances when done** to avoid charges.

**Think about:** your single load balancer is now a single point of failure. How would you remove it?

---

## Quick Revision

- **Functional requirements** = features (what). **Non-functional** = qualities (how well).
- **Availability** = uptime / (uptime + downtime). Each extra nine = 10× less downtime.
- **SPOF** = one part whose failure breaks everything. Fix it with redundancy.
- **Active–active** = all copies serve traffic. **Active–passive** = standby waits.
- **Scalability** = handle more load without getting slower.
- **Vertical** = bigger machine (has limits, SPOF). **Horizontal** = more machines + load balancer.
- A single **load balancer** is also a SPOF.
- **Auto scaling** is too slow for sudden spikes → pre-warm (Hotstar).
- **Load testing** proves scalability — test beyond your expected peak.
- **Maintainability** = clean code structure, monitoring, modular architecture.
- **Fault tolerance** = replication, load balancing, backups, monitoring.
- **Reliability** = works correctly over time. Availability = MTBF / (MTBF + MTTR).
- **S3 2017:** a typo removed too many servers → 4h outage → use limits on tools, multi-region, practice recovery.
- **Hotstar 25M:** pre-warm, test at 2× peak, chaos test, turn off extras under load.
