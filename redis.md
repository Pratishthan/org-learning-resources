# Session 0: Introduction to Redis

## 📘 What is Redis?

**Redis** (REmote DIctionary Server) is an open-source, in-memory data store that can be used as:

- **A Cache**: Store frequently accessed data for ultra-fast retrieval.
- **A Database**: Store rich data structures like strings, hashes, lists, sets, etc.
- **A Message Broker**: Power real-time systems with queues, pub/sub, and streams.

---

## 🚀 Key Characteristics

- **In-Memory**  
  All data is stored in RAM, enabling sub-millisecond response times.

- **Advanced Data Structures**  
  Redis supports more than just key–value pairs:
  - Strings
  - Hashes
  - Lists
  - Sets
  - Sorted Sets
  - Streams
  - Bitmaps
  - HyperLogLogs
  - Geospatial indexes

- **Persistence Options**  
  Although memory-first, Redis can persist data to disk via:
  - **RDB (Redis Database Backups)** — point-in-time snapshots
  - **AOF (Append-Only File)** — logs every write operation

- **High Availability**  
  Redis supports:
  - Replication (master-replica)
  - Sentinel for automated failover
  - Clustering for horizontal scaling

- **Lightweight and Fast**  
  Powered by a single-threaded event loop, Redis delivers **extremely high throughput** with minimal complexity.

---


# 🚀 Setting Up Redis in Node.js

We’ll use [`ioredis`](https://github.com/luin/ioredis) — a robust Redis client suitable for production.

---

## 📦 5.1 Install Dependencies

Initialize a new Node.js project and install `ioredis`:

```bash
npm init -y
npm install ioredis
```
## 🧪 Step 2: Create a Redis Client

Create a new file named `src/test.js` with the following code:
```js
// src/redis.js
const Redis = require('ioredis');

// Create client with lazy connect (connects on first command)
const redis = new Redis({
  host: '127.0.0.1',
  port: 6379,
  lazyConnect: true
});

// Event listeners
redis.on('connect', () => console.log('✅ Redis connected'));
redis.on('error', (err) => console.error('❌ Redis error:', err));

module.exports = redis;
```

---

## 🧪 Step 3: Run Basic Redis Commands

Create a new file named `src/test.js` with the following code:

```js
// src/test.js
const redis = require('./redis');

async function run() {
  // 1. Simple SET and GET
  await redis.set('greeting', 'Hello Redis!');
  const value = await redis.get('greeting');
  console.log('Stored value:', value);

  // 2. Increment a counter
  await redis.set('visits', 0);
  await redis.incrby('visits', 1);
  console.log('Visits:', await redis.get('visits'));

  // 3. Set a key with expiration
  await redis.set('otp:email:alice', '123456', 'EX', 10); // 10 seconds
  console.log('OTP:', await redis.get('otp:email:alice'));

  process.exit(0);
}

run();
```
## Step 4: Run the Script

```bash
node src/test.js
```

## Step 5: Output
```bash
✅ Redis connected
Stored value: Hello Redis!
Visits: 1
OTP: 123456
```

## 🧠 Notes

lazyConnect: true allows the Redis client to delay connecting until a command is issued.

The EX option in redis.set() defines key expiration in seconds.

process.exit(0) ensures the script exits after completing asynchronous operations.


# Redis Core Data Structures - Session 1 

## Table of Contents
- [1.1 Strings](#11-strings-incl-counters--ttl)
- [1.2 Hashes](#12-hashes)
- [1.3 Lists](#13-lists)
- [1.4 Sets & Sorted Sets](#14-sets--sorted-sets)

---

## 1.1 Strings (incl. counters & TTL)

### Purpose
Strings are Redis's simplest data structure, ideal for:
- **Simple caching** - Store any binary-safe data
- **Feature flags** - Boolean or configuration values
- **Counters** - Atomic increment/decrement operations
- **Tokens** - Session tokens, API keys, etc.

### Key Commands

| Command | Description | Example |
|---------|-------------|---------|
| `SET` | Set key to value | `SET user:42 "John"` |
| `GET` | Get value by key | `GET user:42` |
| `INCRBY` | Increment by amount | `INCRBY counter 5` |
| `DECRBY` | Decrement by amount | `DECRBY counter 2` |
| `SETEX` | Set with expiration (seconds) | `SETEX token:abc 3600 "data"` |
| `MGET` | Get multiple keys | `MGET key1 key2 key3` |

### TTL (Time To Live) Semantics

| Command | Description | Example |
|---------|-------------|---------|
| `EXPIRE` | Set expiration in seconds | `EXPIRE user:42 3600` |
| `PEXPIRE` | Set expiration in milliseconds | `PEXPIRE user:42 3600000` |
| `TTL` | Get remaining TTL in seconds | `TTL user:42` |

### Node.js Implementation

```javascript
const redis = require('./redis');

// Store user data with 1-hour expiration
await redis.set('user:42', JSON.stringify({ id: 42, name: 'Ada' }), 'EX', 3600);

// Retrieve and parse user data
const raw = await redis.get('user:42');
const user = JSON.parse(raw);

// Increment page view counter
await redis.incrby('page:home:views', 1);

// Check TTL
const ttl = await redis.ttl('user:42');
console.log(`Expires in ${ttl} seconds`);
```

### Exercise: Profile Cache Implementation

Build a read-through cache that fetches from a mock database on cache miss:

```javascript
class ProfileCache {
  constructor(redis, mockDB) {
    this.redis = redis;
    this.mockDB = mockDB;
    this.TTL = 60; // 1 minute
  }

  async getProfile(userId) {
    const key = `profile:${userId}`;
    
    // Try cache first
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Cache miss - fetch from DB
    const profile = await this.mockDB.fetchUser(userId);
    if (profile) {
      // Store with expiration
      await this.redis.setex(key, this.TTL, JSON.stringify(profile));
    }
    
    return profile;
  }
}
```

---

## 1.2 Hashes

### Purpose
Hashes store field-value pairs within a single key, perfect for:
- **Object storage** - User profiles, product details
- **Database rows** - Efficient representation of structured data
- **Partial updates** - Update specific fields without retrieving entire object

### Key Commands

| Command | Description | Example |
|---------|-------------|---------|
| `HSET` | Set hash field(s) | `HSET user:42 name "Ada" age 30` |
| `HGET` | Get single field | `HGET user:42 name` |
| `HGETALL` | Get all fields | `HGETALL user:42` |
| `HINCRBY` | Increment numeric field | `HINCRBY user:42 visits 1` |
| `HDEL` | Delete field(s) | `HDEL user:42 temp_flag` |

### When to Use Hashes vs JSON Strings

**Use Hashes when:**
- Need to update individual fields frequently
- Want to avoid JSON parsing overhead
- Fields have different access patterns

**Use JSON Strings when:**
- Always read/write entire object
- Complex nested structures
- Need to maintain exact JSON format

### Node.js Implementation

```javascript
// Store user object as hash
await redis.hset('user:42', { 
  id: 42, 
  name: 'Ada', 
  plan: 'pro',
  visits: 0 
});

// Retrieve entire user object
const user = await redis.hgetall('user:42');
console.log(user); // { id: '42', name: 'Ada', plan: 'pro', visits: '0' }

// Update specific field
await redis.hset('user:42', 'last_login', Date.now());

// Increment counter
await redis.hincrby('user:42', 'visits', 1);
```

### Exercise: Rolling User Counters

Implement per-user statistics tracking:

```javascript
class UserStats {
  constructor(redis) {
    this.redis = redis;
  }

  async recordEvent(userId, eventType) {
    const key = `user:${userId}:stats`;
    await this.redis.hincrby(key, eventType, 1);
    
    // Set expiration for cleanup (optional)
    await this.redis.expire(key, 86400); // 24 hours
  }

  async getStats(userId) {
    const key = `user:${userId}:stats`;
    return await this.redis.hgetall(key);
  }

  async recordClick(userId) {
    await this.recordEvent(userId, 'clicks');
  }

  async recordView(userId) {
    await this.recordEvent(userId, 'views');
  }
}

// Usage
const stats = new UserStats(redis);
await stats.recordClick(42);
await stats.recordView(42);
const userStats = await stats.getStats(42);
// Result: { clicks: '1', views: '1' }
```

---

## 1.3 Lists

### Purpose
Lists are ordered collections of strings, excellent for:
- **Queues** - First-in-first-out (FIFO) processing
- **Stacks** - Last-in-first-out (LIFO) processing
- **Recent items** - Activity feeds, notifications

### Key Commands

| Command | Description | Blocking? |
|---------|-------------|-----------|
| `LPUSH` | Push to left (head) | No |
| `RPUSH` | Push to right (tail) | No |
| `LPOP` | Pop from left | No |
| `RPOP` | Pop from right | No |
| `BLPOP` | Blocking pop from left | Yes |
| `BRPOP` | Blocking pop from right | Yes |
| `LRANGE` | Get range of elements | No |
| `LTRIM` | Trim to specified range | No |

### Blocking vs Non-blocking Operations

**Non-blocking**: Returns immediately, even if list is empty
**Blocking**: Waits for specified timeout until element is available

### Node.js Implementation

```javascript
// Add event to queue (left side)
await redis.lpush('events', JSON.stringify({ 
  timestamp: Date.now(), 
  type: 'login',
  userId: 42 
}));

// Process events (blocking pop from right, wait up to 5 seconds)
const result = await redis.brpop('events', 5);
if (result) {
  const [queueName, eventData] = result;
  const event = JSON.parse(eventData);
  console.log('Processing event:', event);
}

// Get recent events without removing them
const recentEvents = await redis.lrange('events', 0, 9); // Latest 10
```

### Exercise: Recent Logins System

Implement a capped list for tracking recent user logins:

```javascript
class RecentLogins {
  constructor(redis, maxItems = 100) {
    this.redis = redis;
    this.maxItems = maxItems;
    this.key = 'recent_logins';
  }

  async addLogin(userId, metadata = {}) {
    const loginData = {
      userId,
      timestamp: Date.now(),
      ...metadata
    };

    // Add to front of list
    await this.redis.lpush(this.key, JSON.stringify(loginData));
    
    // Trim to maintain max size (keep 0 to maxItems-1)
    await this.redis.ltrim(this.key, 0, this.maxItems - 1);
  }

  async getRecentLogins(count = 10) {
    const items = await this.redis.lrange(this.key, 0, count - 1);
    return items.map(item => JSON.parse(item));
  }

  async getLoginCount() {
    return await this.redis.llen(this.key);
  }
}

// Usage
const recentLogins = new RecentLogins(redis);
await recentLogins.addLogin(42, { ip: '192.168.1.1' });
await recentLogins.addLogin(43, { ip: '10.0.0.1' });

const recent = await recentLogins.getRecentLogins(5);
console.log(recent);
```

---

## 1.4 Sets & Sorted Sets

### Sets
Unordered collection of unique strings, perfect for:
- **Membership testing** - Check if item exists
- **Tags/categories** - Store unique labels
- **Set operations** - Union, intersection, difference

#### Set Commands

| Command | Description | Example |
|---------|-------------|---------|
| `SADD` | Add member(s) | `SADD tags:post:9 redis node` |
| `SISMEMBER` | Check membership | `SISMEMBER tags:post:9 redis` |
| `SMEMBERS` | Get all members | `SMEMBERS tags:post:9` |
| `SUNION` | Union of sets | `SUNION tags:post:1 tags:post:2` |
| `SINTER` | Intersection | `SINTER users:online users:premium` |

### Sorted Sets (ZSets)
Ordered collection where each member has a score, ideal for:
- **Leaderboards** - High score rankings
- **Priority queues** - Process by priority/score
- **Time-based data** - Sort by timestamp

#### Sorted Set Commands

| Command | Description | Example |
|---------|-------------|---------|
| `ZADD` | Add with score | `ZADD leaderboard 1500 user:42` |
| `ZSCORE` | Get member's score | `ZSCORE leaderboard user:42` |
| `ZRANGE` | Get range (ascending) | `ZRANGE leaderboard 0 9` |
| `ZREVRANGE` | Get range (descending) | `ZREVRANGE leaderboard 0 9` |
| `ZRANK` | Get rank (0-based) | `ZRANK leaderboard user:42` |

### Node.js Implementation

```javascript
// Working with Sets
await redis.sadd('tags:post:9', 'redis', 'node', 'cache');
const hasRedisTag = await redis.sismember('tags:post:9', 'redis'); // 1 (true)
const allTags = await redis.smembers('tags:post:9');

// Working with Sorted Sets
await redis.zadd('leaderboard:score', 1500, 'user:42');
await redis.zadd('leaderboard:score', 2000, 'user:43');
await redis.zadd('leaderboard:score', 1200, 'user:44');

// Get top 10 with scores
const top10 = await redis.zrevrange('leaderboard:score', 0, 9, 'WITHSCORES');
// Result: ['user:43', '2000', 'user:42', '1500', 'user:44', '1200']
```

### Exercise: Top-10 Leaderboard with Pagination

Implement a comprehensive leaderboard system:

```javascript
class Leaderboard {
  constructor(redis, name) {
    this.redis = redis;
    this.key = `leaderboard:${name}`;
  }

  async addScore(userId, score) {
    // ZADD handles both new entries and updates
    await this.redis.zadd(this.key, score, `user:${userId}`);
  }

  async getTopN(n = 10, withScores = true) {
    const args = withScores ? ['WITHSCORES'] : [];
    const results = await this.redis.zrevrange(this.key, 0, n - 1, ...args);
    
    if (!withScores) return results;

    // Parse scores and userIds
    const leaderboard = [];
    for (let i = 0; i < results.length; i += 2) {
      const userId = results[i].replace('user:', '');
      const score = parseInt(results[i + 1]);
      leaderboard.push({ userId, score, rank: Math.floor(i / 2) + 1 });
    }
    
    return leaderboard;
  }

  async getPage(page = 1, pageSize = 10) {
    const start = (page - 1) * pageSize;
    const end = start + pageSize - 1;
    
    const results = await this.redis.zrevrange(this.key, start, end, 'WITHSCORES');
    const leaderboard = [];
    
    for (let i = 0; i < results.length; i += 2) {
      const userId = results[i].replace('user:', '');
      const score = parseInt(results[i + 1]);
      leaderboard.push({ 
        userId, 
        score, 
        rank: start + Math.floor(i / 2) + 1 
      });
    }
    
    return leaderboard;
  }

  async getUserRank(userId) {
    // ZREVRANK gives 0-based rank in descending order
    const rank = await this.redis.zrevrank(this.key, `user:${userId}`);
    return rank !== null ? rank + 1 : null;
  }

  async getUserScore(userId) {
    const score = await this.redis.zscore(this.key, `user:${userId}`);
    return score ? parseInt(score) : null;
  }

  async getTotalPlayers() {
    return await this.redis.zcard(this.key);
  }

  // Handle ties by getting all users with same score
  async getUsersWithScore(score) {
    return await this.redis.zrangebyscore(this.key, score, score);
  }
}

// Usage Example
const gameLeaderboard = new Leaderboard(redis, 'game:snake');

// Add scores
await gameLeaderboard.addScore(42, 1500);
await gameLeaderboard.addScore(43, 2000);
await gameLeaderboard.addScore(44, 1200);
await gameLeaderboard.addScore(45, 1500); // Tie with user 42

// Get top 5
const top5 = await gameLeaderboard.getTopN(5);
console.log(top5);

// Get page 2 (users ranked 11-20)
const page2 = await gameLeaderboard.getPage(2, 10);

// Check user's rank
const userRank = await gameLeaderboard.getUserRank(42);
console.log(`User 42 rank: ${userRank}`);

// Handle ties
const tiedUsers = await gameLeaderboard.getUsersWithScore(1500);
console.log(`Users with score 1500: ${tiedUsers}`);
```

## Summary

This session covered Redis's four core data structures:

1. **Strings** - Simple key-value storage with TTL support
2. **Hashes** - Field-value pairs for structured data
3. **Lists** - Ordered collections for queues and recent items
4. **Sets & Sorted Sets** - Unique collections and ranked data

Each structure serves specific use cases and offers atomic operations for safe concurrent access. The exercises demonstrate practical patterns commonly used in production applications.




 

## 1. Expiration & Eviction

### 1.1 Purpose

Expiration and eviction mechanisms in Redis serve critical functions:
- **Manage short-lived data** - Sessions, OTPs, temporary cache entries
- **Automatically free memory** - Prevent memory leaks from stale data
- **Control Redis memory footprint** when it reaches `maxmemory` limits

### 1.2 Commands

| Command | Description | Example |
|---------|-------------|---------|
| `SET key value EX <seconds>` | Set with expiration | `SET session:abc "data" EX 3600` |
| `EXPIRE key <seconds>` | Set TTL on existing key | `EXPIRE user:42 1800` |
| `PEXPIRE key <ms>` | Set TTL in milliseconds | `PEXPIRE token:xyz 30000` |
| `TTL key` | Get remaining time-to-live | `TTL session:abc` |
| `PERSIST key` | Remove expiration | `PERSIST user:42` |

### 1.3 Example in Node.js

```javascript
const redis = require('./redis');

async function otpFlow() {
  // Set OTP with 30-second expiration
  await redis.set('otp:user:alice', '123456', 'EX', 30);
  console.log('OTP set');

  // Check remaining TTL
  console.log('TTL:', await redis.ttl('otp:user:alice')); // ~30
  console.log('Value:', await redis.get('otp:user:alice')); // '123456'

  // Wait for expiration
  await new Promise(r => setTimeout(r, 31000)); // wait 31s
  console.log('After expiration:', await redis.get('otp:user:alice')); // null
}

otpFlow();
```

### 1.4 Eviction Policies

When Redis reaches memory limit (`maxmemory`), it uses eviction policies:

| Policy | Behavior |
|--------|----------|
| **noeviction** | Return errors on new writes |
| **allkeys-lru** | Evict least-recently-used key (any key) |
| **volatile-lru** | Evict LRU among keys with expiration only |
| **allkeys-random** | Random eviction from all keys |
| **volatile-random** | Random eviction from keys with expiration |
| **volatile-ttl** | Evict key with shortest remaining TTL |

**Production Note:** Always set TTLs for cache data to avoid memory bloat.

```javascript
// Example: Cache with automatic expiration
class TTLCache {
  constructor(redis, defaultTTL = 3600) {
    this.redis = redis;
    this.defaultTTL = defaultTTL;
  }

  async set(key, value, ttl = this.defaultTTL) {
    const data = typeof value === 'object' ? JSON.stringify(value) : value;
    await this.redis.setex(key, ttl, data);
  }

  async get(key) {
    const value = await this.redis.get(key);
    if (!value) return null;
    
    try {
      return JSON.parse(value);
    } catch {
      return value;
    }
  }

  async extend(key, additionalSeconds) {
    const currentTTL = await this.redis.ttl(key);
    if (currentTTL > 0) {
      await this.redis.expire(key, currentTTL + additionalSeconds);
    }
  }
}
```

---

## 2. Pipelining

### 2.1 Purpose

Redis pipelining optimizes performance by addressing network latency:
- **Redis is single-threaded** - Each command requires a round-trip
- **Pipelining batches commands** into one TCP request - reduces latency
- **Not atomic** (unlike MULTI/EXEC) - commands execute independently

### 2.2 Performance Benefits

- **Reduces network round-trips** from N to 1
- **Improves throughput** for bulk operations
- **Maintains command ordering** without atomicity guarantees

### 2.3 Example in Node.js

```javascript
async function pipelineDemo() {
  const pipeline = redis.pipeline();

  // Queue multiple commands
  for (let i = 1; i <= 5; i++) {
    pipeline.set(`num:${i}`, i);
    pipeline.get(`num:${i}`);
  }

  // Execute all commands at once
  const results = await pipeline.exec();
  console.log(results);
}

pipelineDemo();
```

**Output** (array of [error, result] pairs):
```javascript
[ 
  [ null, 'OK' ], [ null, '1' ], 
  [ null, 'OK' ], [ null, '2' ], 
  [ null, 'OK' ], [ null, '3' ],
  [ null, 'OK' ], [ null, '4' ],
  [ null, 'OK' ], [ null, '5' ]
]
```


### 2.5 Use Cases

- **Bulk inserts/updates** - User profiles, product catalog updates
- **High-throughput logging** - Event tracking, metrics collection
- **Cache warming** - Pre-loading frequently accessed data
- **Batch analytics** - Computing multiple aggregations

---

## 3. Transactions

### 3.1 Purpose

Redis transactions provide atomicity and consistency:
- **Execute multiple commands atomically** - All or nothing execution
- **Ensure consistency** under concurrent access
- **Prevent race conditions** in critical operations

### 3.2 Transaction Commands

| Command | Description |
|---------|-------------|
| `MULTI` | Begin transaction (queue commands) |
| `EXEC` | Execute all queued commands atomically |
| `DISCARD` | Cancel transaction |
| `WATCH key [key ...]` | Monitor keys for changes (optimistic locking) |
| `UNWATCH` | Stop watching all keys |

### 3.3 Basic Transaction Example (MULTI/EXEC)

```javascript
async function atomicTransfer() {
  // Initialize balances
  await redis.set('balance:alice', 100);
  await redis.set('balance:bob', 50);

  // Atomic transfer
  const tx = redis.multi();
  tx.decrby('balance:alice', 30);
  tx.incrby('balance:bob', 30);

  const result = await tx.exec();
  console.log(result); // [ [null, 70], [null, 80] ]
  
  // Verify final balances
  console.log('Alice:', await redis.get('balance:alice')); // 70
  console.log('Bob:', await redis.get('balance:bob'));     // 80
}

atomicTransfer();
```

### 3.4 Optimistic Locking with WATCH

WATCH enables optimistic concurrency control by monitoring keys for changes:

```javascript
async function safeTransfer(from, to, amount) {
  let attempts = 0;
  const maxAttempts = 10;

  while (attempts < maxAttempts) {
    attempts++;
    
    // Watch the source account
    await redis.watch(from);

    // Check balance
    const balance = parseInt(await redis.get(from)) || 0;
    if (balance < amount) {
      await redis.unwatch();
      throw new Error('Insufficient funds');
    }

    // Create transaction
    const tx = redis.multi();
    tx.decrby(from, amount);
    tx.incrby(to, amount);

    // Execute - returns null if watched key was modified
    const result = await tx.exec();
    
    if (result !== null) {
      console.log(`Transfer successful after ${attempts} attempts`);
      return result;
    }
    
    console.log(`Attempt ${attempts} failed, retrying...`);
    // Small random delay to reduce contention
    await new Promise(r => setTimeout(r, Math.random() * 10));
  }
  
  throw new Error('Transfer failed after maximum attempts');
}

// Usage
try {
  await safeTransfer('balance:alice', 'balance:bob', 30);
} catch (error) {
  console.error('Transfer failed:', error.message);
}
```



### 3.6 Use Cases

- **Financial transactions** - Money transfers, payment processing
- **Inventory management** - Preventing overselling, stock reservations
- **User registration** - Ensuring username uniqueness
- **Rate limiting** - Atomic counter increments with time windows
- **Distributed locking** - Coordinating access to shared resources

### 3.7 Transaction Best Practices

1. **Keep transactions short** - Minimize lock time
2. **Handle retries gracefully** - WATCH conflicts are normal
3. **Use exponential backoff** - Reduce contention on retries
4. **Validate before transaction** - Check conditions early
5. **Clean up on failure** - Use UNWATCH and DISCARD appropriately

---

## Summary

This comprehensive guide covered Redis's core data structures and advanced features:

### Core Data Structures:
1. **Strings** - Simple key-value storage with TTL support
2. **Hashes** - Field-value pairs for structured data
3. **Lists** - Ordered collections for queues and recent items
4. **Sets & Sorted Sets** - Unique collections and ranked data

### Advanced Features:
1. **Expiration & Eviction** - Memory management and data lifecycle
2. **Pipelining** - Performance optimization for bulk operations  
3. **Transactions** - Atomicity and consistency for critical operations

Each feature serves specific use cases and provides building blocks for scalable, high-performance applications. The combination of simple data structures with powerful operational features makes Redis an excellent choice for caching, session management, real-time analytics, and distributed system coordination.



## 🧠 Notes
