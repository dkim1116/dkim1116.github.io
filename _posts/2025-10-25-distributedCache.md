---
layout: post
title: "Designing a Distributed Cache System"
---

This is my attempt at thinking through a distributed cache system.

At first this one seemed simpler than some of the other system design problems because the API is tiny. It is basically just `GET`, `SET`, and `DELETE`.

But once I started layering in things like:

* TTL
* eviction
* failover
* sharding
* 1TB of data
* 100k requests/sec

it stopped feeling simple pretty quickly.

These are basically my notes on how I’d reason through it right now.

---

## Requirements

### Non-functional

* Handle up to **1TB of data**
* Handle around **100k requests/sec**
* **Availability > Consistency**
* Low latency for both `GET` and `SET`

---

## Core Entities

The core data is pretty minimal:

* key
* value
* TTL

---

## API

### Set value

```http
POST /:key
{
  "value": "...",
  "ttl": 300
}
```

### Get value

```http
GET /:key
```

### Delete value

```http
DELETE /:key
```

---

## Starting Simple: Single-Node Cache

If I ignore distribution for a second, the cache is basically:

* a hash map for fast lookup
* TTL metadata attached to each entry

### Basic idea

* `SET` stores `key -> value` in the hash map
* if TTL exists, store `expiresAt = now + ttl`
* `GET` checks:
  * does the key exist?
  * if yes, is it expired?
* if expired, delete it and return a miss
* `DELETE` just removes it

Nothing fancy yet.

---

## TTL Design

At first I thought:

> just check expiration on every `GET`

That works, but it is not enough.

The problem is that if a key expires and nobody reads it again, it just sits there forever.

So I’d do both:

### 1. Lazy expiration

* if expired, delete it immediately
* return a miss

### 2. Background cleanup

* periodically scan part of the cache
* remove expired keys

This way:

* hot keys get cleaned up quickly
* cold keys still eventually get removed

---

## Adding LRU Eviction

If memory is limited, we also need a policy for what to evict.

I’d start with **LRU (Least Recently Used)**.

### Data structures

* hash map for fast lookup
* doubly linked list for usage order

### How it works

* every time a key is used, move it to the front
* least recently used items drift toward the tail
* when memory is full, evict from the tail

So the map tells me where the data is, and the list tells me what should get removed.

---

## High-Level Distributed Design

A single node will not handle 1TB or 100k requests/sec.

So I’d split the cache into multiple shards.

Each shard:

* owns part of the keyspace
* has replicas for failover

The next question becomes:

> how do I know which shard a key belongs to?

I’d use **consistent hashing** for that.

---

## Scaling the Cache

To scale this system:

* add more cache nodes
* shard keys using consistent hashing
* replicate each shard

Consistent hashing matters because if I just used:

```text
hash(key) % N
```

then adding one node would reshuffle almost everything.

With consistent hashing:

* only part of the keyspace moves
* scaling becomes a lot less disruptive

---

## Replication and Availability

Since availability matters more than strict consistency here:

* each shard has a primary
* replicas get updated asynchronously

That means:

* writes stay fast
* replicas may lag a little

For a cache, that tradeoff feels reasonable to me.

---

## What Happens If a Node Goes Down?

If a primary goes down, the system needs to:

1. detect the failure
2. check the replicas
3. promote one replica to primary
4. point traffic at the new primary

So the rough flow is:

```text
detect -> promote -> reroute
```

---

## Keeping Latency Low

For a cache, latency is the whole point.

The things I’d pay attention to are:

* keep the cache close to the application servers
* reuse connections
* minimize extra network hops
* keep the hot path fully in memory

If the cache itself is slow, the entire system feels slow.

---

## Rebalancing When Adding a New Node

When adding a new shard:

* consistent hashing determines which keys should move
* existing nodes start copying that data to the new node

During migration:

* some requests may still hit the old node
* the system may need fallback logic while data is still moving

After migration:

* traffic fully shifts to the new node

I would not block reads and writes during migration. That would defeat the point.

---

## How TTL and LRU Work Together

TTL and LRU solve different problems:

* TTL removes data that is no longer valid
* LRU removes data when memory is full

So in my head:

* expired keys should be removed first
* then LRU eviction kicks in if memory pressure still exists

---

## Tradeoffs

This design gives:

* low latency
* horizontal scalability
* good availability

But the tradeoffs are:

* async replication can cause stale reads
* failover is not instant
* rebalancing adds operational complexity
* TTL and eviction both add overhead

---

## What I Learned

* TTL needs both lazy cleanup and background cleanup
* eviction policy is only part of the story, the data structures matter too
* scaling mostly becomes a sharding and routing problem
* failover is more than just adding replicas
* rebalancing is harder than it looks at first

---

## Final Thoughts

This problem looks simple on the surface, but once you start thinking about scale and failure cases, it gets interesting fast.

It was a good reminder for me that even something as basic as a cache turns into a real distributed systems problem once you treat it like production infrastructure.

---

## Future Improvements

* compare LFU vs LRU more deeply
* hot key detection
* multi-region cache design
* persistence / warm restart
* write-through vs write-back
