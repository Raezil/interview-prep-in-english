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
### Load Balancer
- A Kubernetes load balancer service is a component that distributes network traffic across multiple instances of an application running in a K8S cluster
![unnamed](https://github.com/user-attachments/assets/67e3d9eb-0bb5-4b65-9e7a-9a0007181c0f)
	- **Ingress**:
 		- Kubernetes also has an API object called Ingress.
   		- Ingress is built on top of the Kubernetes Service (to expose Ingress, you need to use the Kubernetes Service).
     	- The main responsibility of Ingress is distributing network traffic to services according to predetermined routing rules. 
   
### AWS
https://aws.amazon.com/free/?trk=0a74b2b7-15b3-40f0-a1a9-39d406419e28&sc_channel=ps&ef_id=EAIaIQobChMI-

- **Stripe** handles payments by acting as an intermediary,
	- using a secure gateway to encrypt and transmit payment details from customers to the business's bank account,
	- routing the transaction through the card network (like Visa or Mastercard) and the issuing bank (customer's bank) for authorization
	- then relaying the approval or decline message back to the customer and depositing the funds into the business's bank account after processing fees
