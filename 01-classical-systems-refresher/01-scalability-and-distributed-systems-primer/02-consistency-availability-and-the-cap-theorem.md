# Consistency, Availability, and the CAP Theorem

**Section:** Classical Systems Refresher → Scalability & Distributed Systems Primer | **Est. time:** 2 hrs | **Interview relevance:** High — every AI system design answer that has more than one replica (vector DB, feature store, conversation-state store, RAG index) forces a consistency-vs-availability decision, and interviewers probe whether you can name and defend it.

---

## TL;DR

The CAP theorem says that when a network partition splits your replicas, a distributed data store can keep serving requests (Availability) or guarantee every read sees the latest write (Consistency) — but not both; you must pick one *for that partition*. Partition tolerance is not really a choice — real networks drop packets — so the honest framing is "under a partition, do I sacrifice C or A?" PACELC extends this to the common case: **Else** (no partition), you still trade **L**atency against **C**onsistency on every replicated read. For an Applied AI Engineer this shows up directly: an eventually-consistent RAG index, a strongly-consistent feature store, or a read-your-writes conversation-state store are all CAP/PACELC decisions in disguise. **The one thing to remember: CAP only forces a choice *during a partition* — the everyday, always-present trade-off is latency vs consistency (the "ELC" half of PACELC), and that is what you tune 99% of the time.**

---

## ELI5 — Explain It Like I'm 5

Imagine two shop clerks, one in New York and one in Tokyo, who share a single notebook of prices by phoning each other after every change. Normally when a customer asks a price, a clerk can either answer instantly from their own copy (fast, but maybe just-changed in the other city) or first phone the other clerk to confirm (slow, but definitely correct). Now the phone line between the cities goes dead — this is a "partition." Each clerk must now choose: keep answering customers from a possibly-stale notebook (stay open = Availability) or put up a "closed until the line is back" sign so nobody is ever told a wrong price (stay correct = Consistency). The common misconception is that you get to "pick two of three" like ordering off a menu whenever you like; in reality you only face the hard C-or-A choice *while the line is dead*, and the rest of the time your real decision is just how long you're willing to wait on the phone before answering.

---

## Learning Objectives

By the end of this chapter you will be able to:
- [ ] Define Consistency, Availability, and Partition tolerance using the Gilbert–Lynch/CAP meanings, and explain why "pick 2 of 3" is misleading.
- [ ] Diagnose whether a given data store behaves as CP or AP during a partition, and justify the classification.
- [ ] Apply the PACELC framework to choose a consistency level for an AI component (RAG index, feature store, conversation-state store, agent memory).
- [ ] Compare consistency models (strong / linearizable, bounded staleness, session / read-your-writes, eventual) and select one under a latency and correctness budget.
- [ ] Configure quorum-style read/write consistency (W + R > RF) and explain the availability cost it buys.

---

## Visual Overview

### The CAP Triangle (pick which one to drop under a partition)

```
                 Consistency (C)
                    every read
                  sees latest write
                        ▲
                       ╱ ╲
             CP  ────► ╱   ╲ ◄──── (CA only when NO partition
        (drop A during ╱     ╲        exists — not achievable
         a partition) ╱       ╲       as a real running mode)
                     ╱   pick   ╲
                    ╱   2 of 3   ╲
                   ╱   under a    ╲
                  ╱   partition    ╲
                 ▼──────────────────▼
     Availability (A)            Partition
   every live node responds     tolerance (P)
                    ◄──── AP ────►
              (drop C during a partition)

Reality: P is forced (networks drop packets), so the live
choice is always the C ↔ A edge during a partition.
```

### Partition scenario — CP vs AP behaviour

```
Normal:   Client ──► [Node A] ◄══ replicate ══► [Node B] ──► Client
          (A and B agree; both consistent AND available)

Partition (link A◄══►B is DOWN):

  CP system                         AP system
  ─────────                         ─────────
  Client ──► [Node A]  X  [Node B]  Client ──► [Node A]  X  [Node B]
              │                                 │
              ▼                                 ▼
   "ERROR / timeout: cannot          "here is my local value"
    confirm latest value"             (may be STALE, but responds)
   → sacrifices Availability          → sacrifices Consistency
```

### PACELC decision path

```
Is there a network partition RIGHT NOW?
├── Yes (P) ──► choose:  A  (stay available, allow stale)   → "PA"
│                        C  (return errors, stay correct)   → "PC"
└── No  (E) ──► choose:  L  (answer from one replica, fast) → "EL"
                         C  (quorum/round-trip, correct)    → "EC"

Common real labels:
  PA/EL  = Cassandra, Riak, early Dynamo   (favour uptime + speed)
  PC/EC  = PostgreSQL, VoltDB, Bigtable    (favour correctness always)
  PC/EL  = PNUTS                           (correct under partition,
                                            fast otherwise)
```

---

## Key Concepts

### Consistency (CAP-C / linearizability)

**What is it?** In CAP, consistency means every read receives the most recent completed write or an error — all clients see the same value at the same logical time, regardless of which node they hit. This is *linearizability*, and it is a different, stricter idea than the "C" in ACID (which is about preserving invariants within a transaction).

**How does it work under the hood?** Before a write is acknowledged as successful, it must be propagated (synchronously replicated or committed via a quorum/consensus round) so that any subsequent read on any node cannot observe an older value. That forces at least one network round trip on the write path, and often on the read path too — a read may have to consult a majority of replicas to be sure it isn't reading a stale minority copy.

**Where does it appear in real systems?** Azure Cosmos DB's *Strong* level gives a linearizability guarantee and, in a multi-region account, makes write latency roughly two round trips between the two farthest regions. In DynamoDB you request it per-read with `ConsistentRead=true`; MongoDB exposes it as read concern `"linearizable"` on primary reads. For an AI feature store feeding a fraud model, strong consistency means the score always uses the just-updated feature — no stale-feature false negatives.

### Availability (CAP-A)

**What is it?** In CAP, availability means every request that reaches a *non-failing* node gets a non-error response — but with no promise that the response contains the latest data. Note this is narrower than the colloquial "high availability" / uptime SLA.

**How does it work under the hood?** An AP system lets any reachable replica answer immediately from its local copy without waiting to confirm with peers. During a partition, both sides keep accepting reads (and often writes), then reconcile later using techniques like read-repair, hinted handoff, or anti-entropy Merkle-tree comparison, resolving conflicts (e.g. last-write-wins by timestamp).

**Where does it appear in real systems?** Apache Cassandra with consistency level `ONE` or `LOCAL_ONE` answers from a single replica and stays up even if most replicas are unreachable. DynamoDB's default eventually-consistent read returns whatever a replica has now, at half the cost of a strong read. For a RAG document index, AP behaviour means retrieval keeps serving during a node outage — acceptable because a slightly stale corpus usually beats a downed retriever.

### Partition tolerance (CAP-P) and the forced choice

**What is it?** Partition tolerance is the system's ability to keep operating when an arbitrary number of messages between nodes are dropped or delayed — i.e. when the cluster is split into groups that can't talk.

**How does it work under the hood?** Because real networks *will* drop packets (a partition is indistinguishable from a slow node), a multi-node store must decide, per operation during the split, whether to proceed (risking inconsistency) or to block/error (protecting consistency). This is why CAP is really a two-way choice under P, not a free three-way menu: you don't get to "not tolerate" partitions and still be a distributed system.

**Where does it appear in real systems?** The AWS resilience whitepaper states it plainly: since partitioning must be tolerated, the workload must choose C or A when a partition occurs. Cosmos DB with multiple write regions *cannot* offer Strong consistency precisely because a partition would then force RPO=0 and RTO=0 simultaneously, which is impossible. In practice this is the "single-node databases aren't subject to CAP" insight — CAP bites only once you replicate across a network you don't control.

### Consistency models (the spectrum between strong and eventual)

**What is it?** Consistency is not binary; it's a spectrum of guarantees about *what* a read may observe. Common points: strong/linearizable → bounded staleness → session (read-your-writes / monotonic reads) → consistent prefix → eventual.

**How does it work under the hood?** Weaker levels relax how many replicas must agree before responding. *Session consistency* uses a per-client token: after a write the client caches a session token and passes it on reads, and a replica only answers if it holds data at least as new as that token — guaranteeing you read your own writes without global coordination. *Bounded staleness* caps lag to K versions or T seconds. *Eventual* just returns whatever a replica has, converging later.

**Where does it appear in real systems?** Cosmos DB exposes exactly these five levels as an account setting you can override per request; MongoDB read concern `"majority"` plus a causally-consistent session gives read-your-writes. For agent memory, session/read-your-writes is usually the sweet spot: the agent must see the message it just wrote, but doesn't need a global linear order across all users.

### PACELC (the trade-off that's always on)

**What is it?** PACELC (Abadi, 2010) extends CAP: **if Partition (P)** then choose **A** or **C**; **Else (E)** — normal operation — choose **L**atency or **C**onsistency. It captures the fact that even with no partition, guaranteeing consistency costs a replication round trip, i.e. latency.

**How does it work under the hood?** With no partition, a strongly-consistent read must still contact a quorum (or the leader) to be sure it's current, adding message-delay latency; a low-latency system instead answers from the nearest replica and accepts staleness. So the real, ever-present dial is L↔C, and CAP's C↔A only engages during the (rarer) partition.

**Where does it appear in real systems?** Systems get PACELC labels: Cassandra/Riak/early Dynamo are **PA/EL** (available under partition, low-latency otherwise); PostgreSQL/VoltDB/Bigtable are **PC/EC** (correct always, pay the latency); PNUTS is **PC/EL**. Cosmos DB's five levels are literally a knob spanning the EL↔EC axis. This is the framing to use in interviews because it matches what you actually tune day to day.

### Key Parameters / Configuration Knobs

| Parameter | What it controls | Decision rule |
|---|---|---|
| Replication factor (RF / N) | How many copies of each key exist | Set RF=3 for production durability across failure domains; higher only if you need to survive >1 simultaneous replica loss. |
| Write consistency (W) | How many replicas must ack a write before success | Use `QUORUM`/`majority` when reads must see writes; `ONE`/`LOCAL_ONE` only when write availability and latency beat freshness. |
| Read consistency (R) | How many replicas a read must consult | Ensure `W + R > RF` for read-your-writes; drop to `ONE` for read-heavy, staleness-tolerant paths (e.g. RAG retrieval). |
| Consistency level / read concern | The named freshness guarantee (Strong, Session, Eventual, `majority`, `linearizable`…) | Pick `Session`/read-your-writes for per-user agent state; `Strong`/`linearizable` only for correctness-critical single-item reads (payments, feature scoring). |
| `ConsistentRead` (DynamoDB) | Per-read strong vs eventual | Set `true` only on the read path that must reflect a just-issued write; leave default (eventual) to halve cost and latency elsewhere. |
| Bounded-staleness K / T (Cosmos DB) | Max lag in versions (K) or time (T) | Set the smallest K/T your correctness budget allows; tighter bounds throttle writes when replication lags. |

---

### Worked Example: Requirement → Decision

**Given:** You are building the conversation-state store for a multi-turn LangGraph agent. Each user turn appends a message; the very next node in the graph reads back the conversation to build the LLM prompt. Traffic is global (US + EU regions), p99 read latency budget is 50 ms, and users must *never* see the assistant "forget" the message they just sent.

**Step 1 — Identify the goal:** Guarantee that a client's own just-written turn is visible to that same client's next read (read-your-writes), while keeping reads fast globally. Cross-user global ordering is *not* required.

**Step 2 — Define inputs:** Write ops (append message keyed by `conversation_id`), a per-client/session identity, region of origin, RF=3 within region.

**Step 3 — Define outputs:** A read that returns at least the writer's own latest turn; p99 ≤ 50 ms; stays available if one replica in the region is down.

**Step 4 — Apply constraints:** Strong/linearizable across US↔EU would add a cross-region round trip (>50 ms) — violates latency budget. Pure eventual could drop the just-written turn — violates the "never forget" rule. A partition must not take the whole conversation offline.

**Step 5 — Select the approach:** Use **session / read-your-writes consistency** (Cosmos DB *Session* level, or MongoDB read concern `"majority"` in a causally-consistent session, or Cassandra `LOCAL_QUORUM` with `W + R > RF` so replica sets overlap within the region). Rationale vs alternatives: Session gives the exact guarantee needed (a client sees its own writes) at eventual-consistency-like latency and availability, whereas *Strong* buys global linearizability we don't need at a latency we can't afford, and *Eventual* is too weak because it can lose the writer's own turn. In PACELC terms we chose **PA/EL**-ish with a session token pinning the one guarantee that matters.

---

## Implementation

```python
# Scenario: A fraud-scoring feature store read MUST reflect the feature value
# that was just written by the ingestion step, or the model scores on stale data.
# Correctness beats latency here, so request a strongly consistent read.

import boto3

table = boto3.resource("dynamodb").Table("account_features")

# ConsistentRead=True forces DynamoDB to return the latest committed write,
# reflecting all prior successful writes (a strong read).
resp = table.get_item(
    Key={"account_id": "acct-42"},
    ConsistentRead=True,   # strong read: pay ~2x cost + latency for freshness
)
features = resp["Item"]
score = fraud_model.predict(features)
```

```python
# Anti-pattern: using DynamoDB's DEFAULT (eventually consistent) read for an
# agent's read-your-writes memory. The agent appends a turn, then immediately
# reads the conversation to build the next prompt — but the default read can
# hit a replica that hasn't received the just-written turn, so the agent
# "forgets" what the user just said and produces an incoherent reply.

# WRONG:
table.put_item(Item={"conversation_id": cid, "turn": n, "text": user_msg})
history = table.query(                       # default = eventually consistent
    KeyConditionExpression=Key("conversation_id").eq(cid)
)  # may NOT include the turn we just wrote -> stale prompt

# Correct approach: guarantee read-your-writes on the path that reads back
# your own write. Either request a strong read, or (for cross-region global
# tables) route the read to the same region you wrote to / use a session token.
table.put_item(Item={"conversation_id": cid, "turn": n, "text": user_msg})
history = table.query(
    KeyConditionExpression=Key("conversation_id").eq(cid),
    ConsistentRead=True,   # now the read reflects the write we just made
)
# What breaks without this: the default read trades consistency for lower
# latency/cost (PACELC "EL"); that trade is fine for a RAG corpus but wrong
# for read-your-writes state. Match the consistency level to the guarantee
# the read actually needs, not to a global default.
```

```cql
-- Scenario: A multi-datacenter Cassandra conversation store must let a user
-- read their own writes within their home region without paying a cross-DC
-- round trip. RF=3 per DC; we want the write and read replica sets to overlap.

-- W + R > RF  =>  LOCAL_QUORUM (2) write + LOCAL_QUORUM (2) read, RF=3:
--   2 + 2 = 4 > 3  => the quorums intersect => reads see the latest write.
CONSISTENCY LOCAL_QUORUM;
INSERT INTO conv.messages (conversation_id, turn, text)
  VALUES (?, ?, ?);            -- acked when a local-DC majority responds

CONSISTENCY LOCAL_QUORUM;
SELECT text FROM conv.messages WHERE conversation_id = ?;  -- sees the write
```

---

## Common Pitfalls & Misconceptions

- **"CAP means pick any 2 of 3 whenever you like."** — Beginners read the triangle as a menu and claim things like "we chose CA." In reality partitions are not optional on a real network, so P is a given; the only live choice is C-or-A *during a partition*, and even Brewer later called the "2 of 3" phrasing misleading. Correct model: you sacrifice C or A only while partitioned, and can have both when the network is healthy.
- **"A single-node Postgres is a CP system."** — People label every database with a CAP letter regardless of topology. CAP only applies to data *replicated across a network*; a single node has no partition to tolerate. Correct model: CAP letters describe a *replicated deployment's behaviour under partition*, so classify the cluster, not the product name.
- **"CAP is the trade-off I tune."** — Newcomers obsess over partition behaviour, which is rare, and ignore the constant cost of consistency. The everyday dial is PACELC's "Else" branch: latency vs consistency on normal reads. Correct model: pick the weakest consistency that still meets correctness (often read-your-writes/session), because that's what actually governs p99 latency day to day.
- **"Eventually consistent means eventually correct, so it's fine for agent memory."** — The word "eventually" sounds reassuring, but a read *right after* a write can return the old value or nothing. Correct model: if a step must observe its own prior write (agent state, feature just updated), you need at least read-your-writes/session consistency — eventual is only safe for staleness-tolerant reads like a RAG corpus or like counts.

---

## Key Definitions

| Term | Definition |
|---|---|
| CAP theorem | In a replicated data store, during a network partition you can guarantee Consistency or Availability but not both; partition tolerance is assumed. |
| Consistency (CAP) | Every read returns the most recent completed write or an error — i.e. linearizability; distinct from ACID's "C". |
| Availability (CAP) | Every request to a non-failing node returns a non-error response, without a guarantee of freshness. |
| Partition tolerance | The system keeps operating despite arbitrary loss/delay of inter-node messages. |
| PACELC | Extension of CAP: if Partition → choose Availability or Consistency; Else → choose Latency or Consistency. |
| Linearizability | Strongest consistency: operations appear to take effect instantaneously in a single global order; reads always see the latest committed write. |
| Eventual consistency | Weakest common model: replicas converge to the same value given no new writes, but a read may temporarily return stale data. |
| Session consistency (read-your-writes) | A client is guaranteed to read its own prior writes (and monotonic reads) within a session, without global linear ordering. |
| Quorum rule (W + R > RF) | Choosing write-ack count W and read count R so their replica sets overlap, ensuring reads observe the latest acknowledged write. |
| Bounded staleness | Consistency bounded so replica lag never exceeds K versions or T seconds. |

---

## Summary / Quick Recall

- CAP forces C-vs-A **only during a partition**; P is not optional on a real network.
- "Pick 2 of 3" is the classic trap — Brewer himself walked it back; frame answers as "CP or AP under partition."
- PACELC is the interview-grade framing: the *always-on* trade is Latency vs Consistency (the "ELC" half).
- Consistency is a spectrum: strong/linearizable → bounded staleness → session/read-your-writes → eventual.
- `W + R > RF` (e.g. QUORUM read + QUORUM write, RF=3) buys read-your-writes at the cost of some availability.
- AI mapping: RAG index → AP/eventual is usually fine; feature store for scoring → strong/CP; agent memory & conversation state → session/read-your-writes.
- CAP labels describe a *replicated cluster under partition*, not a product name and not a single node.

---

## Self-Check Questions

1. In the CAP theorem, what does "Consistency" specifically guarantee, and how does it differ from the "C" in ACID?

   <details><summary>Answer</summary>

   CAP consistency means every read returns the most recent completed write or an error (linearizability) across all nodes. It differs from ACID's "C", which is about a single transaction preserving database invariants/constraints, not about all replicas agreeing on the latest value. The tempting wrong answer — "they're the same thing" — fails because ACID-C can hold on a single node with no replication, whereas CAP-C is fundamentally about cross-replica agreement under a network.

   </details>

2. Your team runs a RAG document index across three replicas. During a brief network partition, retrieval should keep returning results even if the corpus is slightly out of date. Which CAP posture and consistency model fit, and why?

   <details><summary>Answer</summary>

   AP with eventual consistency. Retrieval is staleness-tolerant — a slightly outdated corpus still yields useful context — so staying *available* during the partition (answering from a reachable replica) beats erroring out. Choosing CP/strong here would make retrieval return errors during the partition and add latency the rest of the time (PACELC EC) for a guarantee the workload doesn't need.

   </details>

3. A multi-turn agent appends a user's message to a conversation store and, in the next node, reads the conversation back to build the prompt. The store defaults to eventually consistent reads and the agent sometimes "forgets" the latest message. What is the minimal fix?

   <details><summary>Answer</summary>

   Enforce **read-your-writes / session consistency** on that read path — e.g. DynamoDB `ConsistentRead=True`, MongoDB read concern `"majority"` in a causally-consistent session, or Cassandra `LOCAL_QUORUM` reads+writes so `W + R > RF`. This guarantees the client sees its own just-written turn. Jumping straight to global strong/linearizable consistency would also work but is overkill — it adds cross-region latency for a global ordering guarantee the per-user conversation doesn't require.

   </details>

4. **Which TWO** of the following statements about PACELC are correct?
   - A. PACELC only applies when a network partition is occurring.
   - B. In the "Else" (no partition) case, the trade-off is between latency and consistency.
   - C. A PA/EL system favours availability under partition and low latency otherwise.
   - D. PACELC replaces CAP by proving partitions never happen.
   - E. PC/EC systems relax consistency to reduce latency when there is no partition.

   <details><summary>Answer</summary>

   **B and C.** B is the core PACELC insight: even with no partition you still trade latency vs consistency on replicated reads. C correctly describes the PA/EL label (e.g. Cassandra/Riak). A is wrong because PACELC's whole point is the *Else* branch that applies with **no** partition. D is wrong — PACELC extends CAP, it doesn't deny partitions. E is wrong because PC/EC systems (e.g. PostgreSQL, VoltDB) keep consistency in *both* branches and pay the latency; the most tempting distractor is E since it sounds like a normal trade-off, but "EC" means it chooses Consistency in the Else case.

   </details>

5. You must choose between a PC/EC store (e.g. PostgreSQL-style) and a PA/EL store (e.g. Cassandra-style) for the primary feature store that a real-time fraud model reads at scoring time. Under what condition is each the right call, and what's the key risk of the wrong pick?

   <details><summary>Answer</summary>

   Choose **PC/EC** if the model must score on the freshest feature values and a wrong/stale feature causes material harm (missed fraud) — you accept higher latency and reduced availability during partitions to guarantee correctness. Choose **PA/EL** only if slightly stale features are tolerable and scoring throughput/uptime dominate. The key risk of wrongly picking PA/EL for fraud is silent stale-feature scoring (false negatives that look like normal operation); the risk of wrongly picking PC/EC for a staleness-tolerant path is unnecessary latency and lost availability during partitions. The deciding factor is the cost of staleness, not raw performance.

   </details>

---

## Further Reading

- [CAP theorem — AWS resilience whitepaper](https://docs.aws.amazon.com/whitepapers/latest/availability-and-beyond-improving-resilience/cap-theorem.html) — *verified 2026-07-28* — Concise official statement of C/A/P and the forced C-or-A choice under partition.
- [Consistency level choices — Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels) — *verified 2026-07-28* — The five tunable consistency levels, PACELC framing, and quorum/RPO trade-off tables.
- [DynamoDB read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) — *verified 2026-07-28* — Eventually vs strongly consistent reads, the `ConsistentRead` parameter, and global-tables MREC/MRSC modes.
- [Read Concern — MongoDB Manual](https://www.mongodb.com/docs/manual/reference/read-concern/) — *verified 2026-07-28* — `local`/`available`/`majority`/`linearizable`/`snapshot` levels and read-your-writes via causally consistent sessions.
- [Dynamo (architecture) — Apache Cassandra](https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html) — *verified 2026-07-28* — Tunable consistency levels and the `W + R > RF` quorum-overlap rule for read-your-writes.
