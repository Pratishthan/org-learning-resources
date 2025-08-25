Session 0: 

Intro:
  Redis (REmote DIctionary Server) is an open-source, in-memory data store that can be used as:
  A cache: Store frequently accessed data for fast retrieval.
  A database: Keep data structures like strings, hashes, lists, sets, and more.
  A message broker: Implement queues, pub/sub, and streams for real-time systems.
  
Key Characteristics:
  In-Memory → All operations happen in RAM → sub-millisecond response times.
  Data Structures → Not just key–value pairs, but rich structures like Lists, Sets, Sorted Sets, Hashes, Streams, Bitmaps, HyperLogLogs, etc.
  Persistence Options → Though Redis is memory-first, it can persist data to disk (RDB snapshots, AOF logs).
  High Availability → Replication, Sentinel, and Clustering.
  Lightweight → Single-threaded event loop with extremely high throughput.

How Redis Works Internally (High-level mental model)
  Single-threaded event loop → processes commands sequentially, which makes it simple and predictable.
  Data lives in memory → very fast, but limited by available RAM.
  Persistence is optional:
  RDB (snapshotting): saves full dataset at intervals.
  AOF (append-only file): logs every operation, more durable.
  Replication → replica nodes keep copies for HA and scale reads.
  Cluster → sharding across multiple nodes for huge datasets.


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


# Redis Core Data Structures - Session 1 (90 min)

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
