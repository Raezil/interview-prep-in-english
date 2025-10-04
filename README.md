- **Concurrency** is a technique for managing multiple tasks that are executed in an interleaved manner (on a single processor).
- **Parallelism** is executing multiple tasks truly at the same time (on multiple cores or processors).
- The **goroutine** stack weighs at least 2 KB of memory, while OS threads weigh about several MB of memory. The goroutine stack is dynamic and can grow and shrink.

- **Goroutines** communicate with each other via **channels**
  - Channels can be **buffered** or **unbuffered**
    - **Unbuffered** channels enforce a synchronous hand-off (send and receive both block until the other side is ready)
    - **buffered** channels queue up to cap(ch) values so sends block only when full and receives block only when empty.
- The **M:N scheduler** maps **M goroutines** onto **N OS threads**.
  - Each thread has a processor, and that processor has a local queue.
  - If the local queue is empty → the scheduler steals work from other queues to balance the load.
  - If the local queues are full → goroutines go to the global queue..
- **Waitgroup** synchronizes work of many goroutine, It waits for completion every each of them before proceeding further.
- **Mutex** prevents race condition - situation where many goroutines try to access shared variable, try to access critical section of the code.
  - Only one goroutine can access critical section of the code. 
- **sync.RWMutex** is a read–write lock: many goroutines can hold the read lock concurrently (RLock/RUnlock), but the write lock (Lock/Unlock) is exclusive and blocks both readers and writers.

 ### Garbage Collector
- Starts with all objects marked as **white**.  
- **Roots** become **gray**.  
- Each object has a **list of references**.  
- The algorithm traverses the lists:  
  - Objects in the list become **gray**,  
  - When a list has been processed → the object becomes **black**.  
- The process continues until all gray objects become **black**.  
- The remaining white objects are garbage and are **freed from memory**.

### Other Concepts
- An **interface** defines a set of method signatures, **without their implementation**.
- Outside generics, **any** behaves exactly like interface{}:
	- it can hold a value of any dynamic type.
 	- You still need type assertions or type switches to work with the underlying concrete type.
- Can an interface embed another interface in Go?
	- Yes. In Go, an interface may embed zero or more other interfaces. The resulting interface’s method set is the union of all embedded interfaces’ methods
  (duplicates with identical signatures are deduplicated).
	- A type implements the composed interface if it implements all methods from that union.
- A **pointer** refers to a **location in memory**.
- In Go, everything is passed to functions **by value**.
	- That means you always copy the argument.
	- If the argument is a "handle" (e.g., a pointer, slice, map, channel, function, or interface),
 		- you copy the handle itself, but it still points to the same underlying memory.


### Generics in Go
Since **Go 1.18**, the language supports **generics**.  
They let you write functions and structs that operate on different types **without duplicating code**.

### Panic and Recover
- `panic` stops the normal flow of the program.  
- `recover` inside a `defer` catches the panic and lets you regain control.  

**Defer** delays the execution of a function until the surrounding function returns.  
  - It operates in **LIFO** (*Last In – First Out*) order.  

### Context
- Used to propagate timeouts, deadlines, cancellation, and metadata across goroutines.
- The standard mechanism in Go for concurrency control.

### Struct Embedding
- Go doesn’t have classical inheritance, but it supports composition.
- Embedding lets you use the fields and methods of the embedded struct directly.

### Slice
- A slice is a dynamic view over an array — it has length and capacity.
- It can grow using `append`.

### Map
- A map is a key → value dictionary.
- It is not concurrency-safe — use `sync.Mutex` or `sync.Map`.

### HTTP Methods
- **GET / HEAD** — read-only; does not change state. `GET` returns the body; `HEAD` returns headers only.
- **POST** — create/action; the server assigns an ID or there are side effects.
- **PUT** — you know the resource URI; replace the entire resource (or upsert).
- **PATCH** — modify part of the resource.
- **DELETE** — remove a resource; repeating the call is safe (idempotent).
- **OPTIONS** — CORS/preflight/discover supported methods.


### gRPC: How It Works (Summary)
- **IDL → code:** Define the API in `.proto`. `protoc` generates code in your chosen language (here: Go).
- **Server:** The generator creates a server interface; you implement it, usually by embedding `Unimplemented<YourService>Server` to preserve backward compatibility when adding methods.
- **Client:** The generator creates a client stub; you call gRPC methods like regular functions — unary and streaming (server, client, bidirectional).
- **Transport:** HTTP/2, binary Protobuf, and a strongly typed contract on both sides.

## gRPC Streaming Overview

* **Transport:** gRPC rides **HTTP/2**. Each RPC is a single HTTP/2 **stream** with: **headers (metadata) → messages (length‑prefixed protobuf frames) → trailers (status)**.

* **RPC shapes:**

  * **Unary:** 1 req → 1 res
  * **Server‑streaming:** 1 req → many res
  * **Client‑streaming:** many req → 1 res
  * **Bidirectional:** many ↔ many (full‑duplex)

* **Flow control / backpressure:** handled by **HTTP/2 windows**; in Go you feel it as blocking on `Send`/`Recv`.

* **Cancellation / deadlines:** propagate via `context.Context`.

* **Ordering:** messages are **in‑order** within an RPC. The **first error closes the stream**.



### Cache
- A cache is a small, fast key→value store you put in front of a slower system (disk, DB, network, compiler, etc.) to serve repeated reads quicker.
```go
package cache

import (
	"sync"
	"time"
)

// Option configures the Cache at construction time.
type Option func(*cacheConfig)

type cacheConfig struct {
	defaultTTL     time.Duration // 0 => no default expiry
	cleanupEvery   time.Duration // 0 => disable background janitor
}

func WithDefaultTTL(ttl time.Duration) Option {
	return func(c *cacheConfig) { c.defaultTTL = ttl }
}

func WithCleanupInterval(every time.Duration) Option {
	return func(c *cacheConfig) { c.cleanupEvery = every }
}

type entry[V any] struct {
	value    V
	expireAt time.Time // zero => no expiry
}

// Cache is a basic in-memory key-value cache with optional TTL.
type Cache[K comparable, V any] struct {
	mu           sync.RWMutex
	items        map[K]entry[V]
	defaultTTL   time.Duration
	cleanupEvery time.Duration
	stopCh       chan struct{}
	doneCh       chan struct{}
}

// New creates a new Cache instance.
// Example: New[string,int](WithDefaultTTL(time.Minute), WithCleanupInterval(30*time.Second))
func New[K comparable, V any](opts ...Option) *Cache[K, V] {
	cfg := cacheConfig{}
	for _, o := range opts {
		o(&cfg)
	}
	c := &Cache[K, V]{
		items:        make(map[K]entry[V]),
		defaultTTL:   cfg.defaultTTL,
		cleanupEvery: cfg.cleanupEvery,
	}
	if c.cleanupEvery > 0 {
		c.startJanitor()
	}
	return c
}

// Close stops the background janitor (if enabled).
func (c *Cache[K, V]) Close() {
	if c.stopCh == nil {
		return
	}
	close(c.stopCh)
	<-c.doneCh
}

// Set adds or updates a key-value pair using the cache's default TTL (if any).
func (c *Cache[K, V]) Set(key K, value V) {
	c.SetWithTTL(key, value, c.defaultTTL)
}

// SetWithTTL adds or updates a key-value pair with a specific TTL.
// ttl <= 0 means no expiration for this entry.
func (c *Cache[K, V]) SetWithTTL(key K, value V, ttl time.Duration) {
	var exp time.Time
	if ttl > 0 {
		exp = time.Now().Add(ttl)
	}

	c.mu.Lock()
	c.items[key] = entry[V]{value: value, expireAt: exp}
	c.mu.Unlock()
}

// Get retrieves the value by key. If the item is expired, it is removed and (zero,false) is returned.
func (c *Cache[K, V]) Get(key K) (V, bool) {
	// Fast path under read lock.
	c.mu.RLock()
	e, ok := c.items[key]
	if !ok {
		c.mu.RUnlock()
		var zero V
		return zero, false
	}
	expired := !e.expireAt.IsZero() && time.Now().After(e.expireAt)
	value := e.value
	c.mu.RUnlock()

	if !expired {
		return value, true
	}

	// Upgrade to write lock to delete if still expired.
	c.mu.Lock()
	if e2, ok2 := c.items[key]; ok2 && !e2.expireAt.IsZero() && time.Now().After(e2.expireAt) {
		delete(c.items, key)
	}
	c.mu.Unlock()

	var zero V
	return zero, false
}

// Remove deletes the key-value pair with the specified key from the cache.
func (c *Cache[K, V]) Remove(key K) {
	c.mu.Lock()
	delete(c.items, key)
	c.mu.Unlock()
}

// Pop removes and returns the value associated with the key (if present and not expired).
func (c *Cache[K, V]) Pop(key K) (V, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	e, ok := c.items[key]
	if !ok {
		var zero V
		return zero, false
	}

	if !e.expireAt.IsZero() && time.Now().After(e.expireAt) {
		delete(c.items, key)
		var zero V
		return zero, false
	}

	delete(c.items, key)
	return e.value, true
}

// ----- internals -----

func (c *Cache[K, V]) startJanitor() {
	c.stopCh = make(chan struct{})
	c.doneCh = make(chan struct{})

	go func() {
		defer close(c.doneCh)
		t := time.NewTicker(c.cleanupEvery)
		defer t.Stop()

		for {
			select {
			case <-t.C:
				c.removeExpired()
			case <-c.stopCh:
				return
			}
		}
	}()
}

func (c *Cache[K, V]) removeExpired() {
	now := time.Now()
	c.mu.Lock()
	for k, e := range c.items {
		if !e.expireAt.IsZero() && now.After(e.expireAt) {
			delete(c.items, k)
		}
	}
	c.mu.Unlock()
}
```

### SQL join
![1m55Wqo](https://github.com/user-attachments/assets/04e9fd01-15eb-4cec-8f94-21ad7a683ba9)

### Autoscaling
- Horizontal autoscaling (HPA/KEDA): add/remove pods.
- Vertical autoscaling (VPA): resize each pod’s CPU/memory.


### AWS
https://aws.amazon.com/free/?trk=0a74b2b7-15b3-40f0-a1a9-39d406419e28&sc_channel=ps&ef_id=EAIaIQobChMI-

- **Stripe** handles payments by acting as an intermediary,
	- using a secure gateway to encrypt and transmit payment details from customers to the business's bank account,
	- routing the transaction through the card network (like Visa or Mastercard) and the issuing bank (customer's bank) for authorization
	- then relaying the approval or decline message back to the customer and depositing the funds into the business's bank account after processing fees


# MySQL / MariaDB, AWS, Redis, and Resilient Stripe Integrations — Interview Q&A

---

## MySQL / MariaDB (backend-heavy Q&A)

**1) How do you choose indexes?**
**A:** Use workload-driven indexing. Prefer composite indexes that match query predicates’ left-to-right order; cover queries (include SELECT columns in the index) to avoid back-to-table lookups; avoid redundant/overlapping indexes; measure via `EXPLAIN`, `rows`, `type`, `filtered%`, `Extra: Using index`.

**2) Composite index order?**
**A:** Put the most selective, equality-filtered columns first, then range columns. Leverage the **leftmost prefix** rule.

**3) InnoDB isolation levels & phantoms**
**A:** Defaults to **REPEATABLE READ** (InnoDB MVCC). Non-locking “consistent reads” see a snapshot. Locking reads (`FOR UPDATE`/`LOCK IN SHARE MODE`) use **next-key locks** (record + gap) to prevent phantoms. At **READ COMMITTED** fewer gap locks → less contention, but you must handle phantoms at app level.

**4) Deadlocks: how do you prevent & react?**
**A:** Keep transactions short; access rows in a consistent order; use correct indexes (to lock as few rows as possible); prefer RC if safe; on error 1213/40001 **retry with jitter**.

**5) Online schema changes, zero downtime**
**A:** Prefer **ONLINE/INPLACE/INSTANT** DDL when supported (MySQL 8 INSTANT for some ops). Otherwise gh-ost/pt-osc with dual-write or triggers; deploy in phases; backfill; cut over with minimal lock.

**6) Reads from replicas without stale data bugs?**
**A:** Read-your-writes by: stickiness per session/user; route critical reads to primary; or use GTID/last-committed LSN fences (wait until replica ≥ timestamp/GTID).

**7) MySQL vs MariaDB you should know**
**A:** Mostly compatible, but differences in GTIDs, some optimizer/DDL features, and replication/cluster tech (e.g., MySQL Group Replication vs MariaDB Galera). Validate feature parity before cross-migrating.

**8) AUTO_INCREMENT contention under load**
**A:** Understand `innodb_autoinc_lock_mode`; batch inserts & surrogate keys can serialize. Consider UUIDv7/KSUID for sharding/hot-spot relief (mind index locality).

**9) Query gotcha you’ve fixed before**
**A (pattern):** “SELECT … WHERE function(col)=…”. Move function to the RHS or precompute to keep index usable.

**10) Backups & PITR**
**A:** Physical snapshots (XtraBackup/RDS snapshots) + binary logs for **point-in-time restore**; test restores regularly; encrypt keys in KMS.

---

## AWS (what interviewers love to probe)

**1) Sketch a highly available web stack**
**A:** VPC (private subnets), ALB → ECS/Fargate or EKS → app. RDS/Aurora (Multi-AZ), ElastiCache Redis, S3, SQS. NAT for egress, SGs for L4 allow-lists, WAF on ALB/CloudFront, CloudWatch/X-Ray, IAM least privilege, Secrets Manager/KMS.

**2) ALB vs NLB?**
**A:** ALB = L7, path/host routing, WebSockets/HTTP2. NLB = L4, extreme throughput, static IPs, TLS passthrough, low latency.

**3) S3 consistency & common patterns**
**A:** Strong read-after-write for puts/overwrites now. Use S3 + CloudFront for static; multipart for large; lifecycle to Glacier; SSE-KMS for encryption.

**4) SQS Standard vs FIFO**
**A:** Standard: at-least-once, best-effort ordering, high throughput. FIFO: ordered, exactly-once (with dedup), lower throughput; use message groups for parallelism.

**5) RDS/Aurora resilience**
**A:** Multi-AZ for automatic failover; automated & manual snapshots; cross-region replicas for DR. Monitor replica lag; plan RPO/RTO.

**6) IAM least privilege in practice**
**A:** Role-based access; scoped policies; temporary creds via STS; customer-managed keys in KMS; rotate secrets in Secrets Manager.

**7) Blue/Green or Canary on AWS**
**A:** ECS+CodeDeploy/ALB weighted target groups or Lambda aliases; health probes; automated rollback on alarms.

**8) Cost levers you actually use**
**A:** Graviton, Savings Plans/RI, right-size/ECS Fargate Spot, S3 lifecycle, compress logs, turn on autoscaling based on SQS depth/CPU.

---

## Redis (not “just a cache”)

**1) Data structures & when to use**
**A:** Strings (counters/rate limits), Hashes (profiles/config), Sets/Sorted Sets (tags, leaderboards), Lists/Streams (queues/logs), Bitmaps/HyperLogLog (flags/approx counts).

**2) Persistence & durability**
**A:** RDB (snapshot) vs AOF (append-only). For critical data: AOF + fsync policy, or treat Redis as cache only with durable system of record (MySQL).

**3) Eviction & memory pressure**
**A:** Policies: noeviction, allkeys-lru/mfu/random, volatile-*. Size with headroom; monitor `maxmemory`. Use TTLs and avoid unbounded keys.

**4) Cache stampede (“thundering herd”)**
**A:** Singleflight/locks; **stale-while-revalidate**; probabilistic early expiration; jittered TTLs; local LRU cache in process.

**5) Distributed lock caveats**
**A:** Redlock exists but has tradeoffs; prefer sharding-safe idempotency and transactional outbox where possible; if you lock, set short TTL, renew, and design for lock loss.

**6) Redis Cluster vs Sentinel**
**A:** Sentinel = HA for a single primary (no sharding). Cluster = sharding across hash slots + HA; multi-key ops must target same slot or use Lua.

**7) Rate limiting in practice**
**A:** `INCR` + `EXPIRE`, or token bucket with `SETNX`/`PTTL`, or Lua for atomicity; log to metrics; return `Retry-After`.

---

## Resilient Stripe Integrations (payments are failure-rich)

**1) What API flow do you prefer?**
**A:** **Payment Intents** (handles SCA/3DS). Client collects payment method → server creates/updates Intent → client confirms → server verifies final status via **webhook**.

**2) Idempotency: where and how?**
**A:** Use Stripe’s `Idempotency-Key` on POSTs (unique per logical action). Also implement **app-level idempotency** (table keyed by `(action_key → status, result, checksum)`), so retries or duplicates don’t double-charge or double-fulfill.

**3) Webhooks: safe processing**
**A:** Verify signature (tolerate clock skew); **ack fast** (2xx) after persisting to an **inbox table/queue** (SQS). Process asynchronously, **idempotently** keyed by `event.id`. Expect retries/out-of-order delivery.

**4) Exactly-once fulfillment?**
**A:** Use **transactional outbox/inbox**: DB transaction writes your business state + an outbox row; a worker publishes/executes side effects. Consumers enforce idempotency with a processed-events table.

**5) Reconciliation**
**A:** Periodic backfill using Stripe’s balance transactions/payouts; compare against your ledger; alert on mismatches; maintain an immutable **double-entry ledger** for internal balances.

**6) Partial failures & timeouts**
**A:** Retries with exponential **jitter**; circuit breaker around Stripe; DLQ for poison messages; compensating actions (refund/void) if downstream step fails.

**7) Security & key hygiene**
**A:** Restrict API keys; store in Secrets Manager; rotate; use **Ephemeral Keys** for mobile; scope Connect keys per account if using platforms.

**8) Multi-currency & rounding**
**A:** Store amounts as **integers in minor units**; never float. Be explicit about currency exponent; display formatting is a UI concern.

---

## System-Design Scenarios (practice out loud)

**1) Design a robust order-checkout with Stripe, MySQL, Redis, AWS**

* **API** behind ALB/ECS. **MySQL** is source of truth. **Redis** for sessions, product cache, and rate limits.
* On “Place Order”: start DB TX → reserve inventory → create Payment Intent with idempotency key → commit order in `pending_payment` + write **outbox** record.
* Outbox worker calls Stripe (retries/jitter). **Webhook** (via API Gateway → SQS) verifies signature, enqueues event. Worker marks order `paid` idempotently and publishes “order_paid”.
* Reads can go to **replica** except “my last order” (stick to primary).
* Observability: structured logs, metrics (authorization rate, webhook retry rate, time-to-paid), trace Stripe calls.
* Failure drills: duplicate webhooks, Stripe outage, DB deadlock, SQS DLQ processing.

**2) Hot product page gets hammered**

* Redis cache with SWR + singleflight; coalesce misses; per-IP rate limit; precache on change; TTL jitter; fallback to last-known-good.
