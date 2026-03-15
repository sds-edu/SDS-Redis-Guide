# SDS Toolbox: Redis In-Memory Datastore Guide

## Introduction

The Software Design School (SDS) Toolbox is a collection of guides and resources to help you get started with the various tools and technologies used in software engineering.

In this guide, we will explore how we can introduce an in-memory datastore to buffer the load and protect our system.

### What is Redis?

**Redis** (REmote DIctionary Server) is an open-source, in-memory data structure store. Because it keeps all data in RAM rather than writing it immediately to a spinning disk or SSD, operations are incredibly fast—often reading and writing in under a millisecond.

While we are using it as a high-speed buffer today, Redis is a versatile tool in modern software engineering. Common use cases include:

* **Caching:** Storing the results of expensive database queries or API calls to serve subsequent requests instantly.
* **Session Management:** Storing user login sessions in a distributed system where multiple backend servers need access to the same session state.
* **Pub/Sub Messaging:** Acting as a message broker for real-time chat applications or event-driven microservices.
* **Leaderboards:** Using its native "Sorted Sets" data structure to instantly rank millions of users by score in real-time.
* **Rate Limiting:** Tracking how many API requests a specific IP address has made in the last minute.

### How Redis Differs From Traditional Databases

Relational databases like PostgreSQL and document stores like MongoDB focus on durable, queryable storage with rich query languages. Redis focuses on:

* Speed: most operations are O(1) or close to it, and run in memory.
* Simplicity: operations are on simple data structures, not joins or complex queries.
* Ephemeral use cases: caching, real-time coordination, transient queues.

We typically still keep long-term data (e.g., users, history) in a “real” database and use Redis for fast, derived, or ephemeral state.

---

## 1. Getting Started

The simplest way to install Redis is usually Docker (assuming Docker Desktop is installed). You can pull the official Redis image and run it with this command:

```bash
docker run -d --name redis -p 6379:6379 redis:latest

```

This starts a Redis server listening on port 6379 on your machine.

You can interact with it from the command line using the Redis CLI (inside the container):

```bash
docker exec -it redis redis-cli

```

Inside `redis-cli`, you can type commands like:

```bash
SET mykey "Hello\nWorld"
GET mykey

```

---

## 2. Understanding Keys

It is important to understand how Redis identifies data. Everything in Redis is stored as a **Key-Value pair**.

* **The Key:** This is the unique identifier you use to store and retrieve your data. Redis keys are "binary safe" (meaning a key could technically be anything from a simple text string to the raw bytes of a JPEG file), but they are almost always standard text strings.
* **The Value:** This is the actual data payload. Unlike simple caches that only store basic strings, Redis values can be complex data structures (which we will explore in the next section).

### Key Naming Best Practices

Because Redis doesn't have "tables" or "columns" like a relational database, the key name itself does all the heavy lifting for organizing your data.

* **Use Colons for Namespacing:** The universal convention is to use colons (`:`) to separate categories, moving from broad to specific. For example, `user:1001:email` or `article:9938:comments`.
* **Size Limits:** While the maximum allowed size for a single key in Redis is a massive 512 MB, typical keys should just be a few easily readable bytes.

To explore more, see the Redis documentation on [keys](https://redis.io/docs/latest/develop/using-commands/keyspace/).

---

## 3. Core Data Structures & Operations

Unlike traditional databases that use tables or documents, Redis is a *data structure server*. You choose the structure that best fits your access pattern.

### 3.1 Strings

The most basic Redis type, representing a sequence of bytes. They are ideal for simple key-value caching, session tokens, or atomic counters.

* `SET key value`: Sets the value of a key.
* `GET key`: Retrieves the value.
* `INCR key`: Increases an integer value by one. This operation is atomic, making it perfect for tracking page views or rate limiting.
* `MSET key1 value1 key2 value2`: Sets multiple keys in a single operation to save network round-trips.
* `MGET key1 key2`: Retrieves multiple keys at once.

```bash
> INCR page_views:home
(integer) 1
> INCR page_views:home
(integer) 2
> MSET color:1 "Red" color:2 "Blue" color:3 "Green"
OK
> MGET color:1 color:2 color:3
1) "Red"
2) "Blue"
3) "Green"
```

### 3.2 Hashes

Hashes are collections of field-value pairs. They are perfect for representing objects without needing to serialize/deserialize JSON strings.

* `HSET key field value [field value ...]`: Sets one or more fields in the hash.
* `HGET key field`: Gets the value of a specific field.
* `HGETALL key`: Returns all fields and values in the hash.

```bash
> HSET user:1234 username "johndoe" status "online"
(integer) 2
> HGET user:1234 username
"johndoe"
> HGETALL user:1234
1) "username"
2) "johndoe"
3) "status"
4) "online"
```

### 3.3 Lists

Lists are linked lists of strings. Because they preserve the order of insertion and offer constant time O(1) inserts at the top and bottom, they are excellent for implementing job queues, recent event logs, or buffering high-velocity write loads.

* `LPUSH key value`: Pushes an element to the head (left) of the list.
* `RPUSH key value`: Pushes an element to the tail (right).
* `LPOP key`: Removes and returns the first element.
* `LRANGE key start stop`: Returns elements in a specified range. Using `0 -1` fetches the entire list.

```bash
> RPUSH write_buffer "payload_A"
(integer) 1
> RPUSH write_buffer "payload_B"
(integer) 2
> LPUSH write_buffer "payload_C"
(integer) 3
> LPOP write_buffer
"payload_C"
> LRANGE write_buffer 0 -1
1) "payload_A"
2) "payload_B"
```

### 3.4 Sets

Sets are unordered collections of unique strings. They are ideal for tracking presence (e.g., which users are currently in a session) or any scenario where you need to ensure uniqueness without caring about order.

* `SADD key member`: Adds one or more members to a set.
* `SMEMBERS key`: Returns all members in the set.
* `SISMEMBER key member`: Checks if a value exists in the set (returns `1` if true, `0` if false).

```bash
> SADD user:1234:roles "admin" "editor"
(integer) 2
> SMEMBERS user:1234:roles
1) "admin"
2) "editor"
> SISMEMBER user:1234:roles "admin"
(integer) 1
> SISMEMBER user:1234:roles "viewer"
(integer) 0
```

### 3.5 Sorted Sets

Similar to Sets, but every member is associated with a floating-point "score". Elements are kept ordered by this score. This is the industry-standard structure for building real-time leaderboards, etc.

* `ZADD key score member`: Adds members with their associated scores.
* `ZRANGE key start stop WITHSCORES`: Returns members in ascending order.
* `ZREVRANGE key start stop WITHSCORES`: Returns members in descending order (e.g., highest score first).

```bash
> ZADD game:leaderboard 100 "PlayerA" 200 "PlayerB"
(integer) 2
> ZRANGE game:leaderboard 0 -1 WITHSCORES
1) "PlayerA"
2) "100"
3) "PlayerB"
4) "200"
> ZREVRANGE game:leaderboard 0 -1 WITHSCORES
1) "PlayerB"
2) "200"
3) "PlayerA"
4) "100"
```

To explore more, see the Redis documentation on [data types](https://redis.io/docs/latest/develop/data-types/).

---

## 4. Key Management & Expiration (TTL)

When using Redis as a cache or a temporary buffer, you rarely want data to live forever and consume all your RAM. Redis provides built-in Time-To-Live (TTL) functionality.

* `EXPIRE key seconds`: Sets a timeout on a key. Once the timeout expires, Redis automatically deletes the key.
* `TTL key`: Returns the remaining time to live of a key (in seconds).
* `DEL key`: Manually deletes one or more keys.
* `EXISTS key`: Checks if a key currently exists.

```bash
> SET mylock "locked"
OK
> EXPIRE mylock 10
(integer) 1
> TTL mylock
(integer) 8
```

Try fetching the key after 10 seconds:

```bash
> GET mylock
(nil)
```

Exiting the CLI is done with executing the `EXIT` command:

```bash
exit
```

---

## 5. Connecting to an Application

To tie concepts together into a single application we will implement a live application called **CompUni Live Q & A**. This app will allow students to submit questions during a lecture, upvote each other's questions, and see live updates on the most popular questions.

Imagine 500 students in a lecture hall. They are joining the room, submitting questions, and rapidly upvoting each other's questions. We will use Redis to buffer, queue, and broadcast this data.

![CompUni Page](./images/compuni%20page.png)

### Step 1: Getting Started

Clone the starter repository [SDS-Kit-Redis](https://github.com/sds-edu/SDS-Kit-Redis), install the dependencies, and start your local Redis and SQLite environment:

```bash
git clone https://github.com/sds-edu/SDS-Kit-Redis
cd SDS-Kit-Redis
npm install
docker-compose up -d
npm start
```

Open `server.js`. You will find several `TODO` comments. Let's build out our Redis application step-by-step.

---

### Part 1: Room Setup & Active Presence (Hashes & Sets)

To launch a session, we need a way to store room details and track unique participants in real time. We’ll use Hashes to group room information into a single object and Sets to ensure our list of active users remains unique and easy to query.

Locate the `TODO: [Room Creation]` section. We will use a **Hash** to store the room info.

Implement the following code:

```javascript
    await redis.hSet(`room:${roomId}:meta`, {
        topic,
        speaker,
        status: 'active'
    });
```

Locate the `TODO: [Heartbeat & Sessions]`. We will use a **Set** to track active users in the room, and a simple `SET` with expiration to track individual user sessions.

Implement the following code:

```javascript
    await redis.sAdd(`room:${roomId}:active_users`, userId);
    await redis.set(`session:${userId}`, 'active', { EX: 15 });
```

---

### Part 2: The Question Queue (Lists)

Questions need to be handled in the exact order they arrive to keep the flow fair. Redis Lists are the ideal tool for this, allowing us to build a high-speed, First-In-First-Out (FIFO) queue that handles high concurrency with ease.

Locate `TODO: [Ask Question]` section. We will use a **List** to implement this queue. Each question will be stored as a JSON string that comes from the request body.

Implement the following code:

```javascript
    await redis.rPush(`room:${roomId}:question_queue`, JSON.stringify(question));
```

---

### Part 3: Atomic Operations (Transactions)

When a speaker clicks "Answer Next Question", we have to update the queue and the global statistics simultaneously.

1. The question is popped from the front of the queue.
2. The "Total Questions Answered" statistic is incremented.

A server hiccup between step 1 and 2 could result in corrupted statistics, so we need to ensure these operations happen **atomically**.

Locate the `TODO: [Answer Next Question]` section. We will use a Redis **Transaction** to ensure these two operations happen together. The `multi()` method allows us to queue up multiple commands and execute them atomically with `exec()`.

Implement the following code:

```javascript
    const [poppedQuestion, totalAnswered] = await redis.multi()
        .lPop(`room:${roomId}:question_queue`)
        .incr('stats:total_answered')
        .exec();
```

---

To fix this section of the guide, we need to ensure the explanation accurately reflects the "Source of Truth" code you are now using (which uses `dirty_questions` and `upvotes`) rather than the old "Buffer" code (which used `upvotes_buffer`).

Here is the corrected **Part 4** for your guide:

---

Here is a revised version of that section tailored for undergrads. I’ve fixed the mismatched bullet points (since your code upgraded from the `dummyId` buffer to the much smarter `dirty_questions` Set) and added the prompt to check the terminal.

---

### Part 4: The Upvote (Buffering & Eventual Consistency)

High-frequency events like "upvote spam" can easily overwhelm a traditional disk-based database like SQLite, leading to "Database Locking" errors and a poor user experience.

To solve this, we use an **Eventual Consistency** pattern. We will use Redis to instantly tally the votes in memory, and a background worker to sync those final totals to SQLite in batches.

#### 1. The Upvote Route

Locate the `TODO: [Upvote Question]` section. Each time an upvote comes in, we will increment the counter in Redis *instead* of hitting SQLite. We also need to keep track of *which* questions received votes so our worker knows what to update later.

Implement the following code:

```javascript
    //  Instantly increment the total upvote count in memory
    await redis.incr(`question:${questionId}:upvotes`);

    // Add the question ID to a "dirty" set to flag it for the background worker
    await redis.sAdd('dirty_questions', questionId);

```

#### 2. The Background Worker

Locate the `TODO: [Eventual Consistency Worker]` section. Every 10,000 milliseconds (10 seconds), we will run a background job that checks the `dirty_questions` Set for any questions that received new upvotes, and safely writes them to SQLite.

Implement the following code:

```javascript
    // Get only the IDs of questions that actually had new upvotes
    const questionIds = await redis.sMembers('dirty_questions');

    for (const questionId of questionIds) {
        // Get the current TOTAL from Redis
        const totalVotes = await redis.get(`question:${questionId}:upvotes`);

        if (totalVotes) {
            // Update SQLite to MATCH the Redis total
            db.run("UPDATE questions SET upvotes = ? WHERE id = ?", [totalVotes, questionId], (err) => {
                if (!err) {
                    // If sync succeeded, remove from the "dirty" set
                    redis.sRem('dirty_questions', questionId);
                    console.log(`Synced total ${totalVotes} votes for ${questionId}`);
                }
            });
        }
    }

```

* **`redis.incr(...)`**: Redis Strings handle atomic increments perfectly. Even if 1,000 requests hit at the exact same millisecond, Redis accurately counts every single one without race conditions.
* **`redis.sAdd(...)`**: Redis Sets only store *unique* values. Even if a question gets 500 upvotes, its ID is only added to this list once!
* **`db.run("UPDATE...")`**: This is where we save our database. Instead of 500 individual disk writes, SQLite performs just **one** efficient update to save the batch of votes.
* **`redis.sRem(...)`**: Once successfully saved to SQLite, we remove the ID from the "dirty" set so we don't unnecessarily ping SQLite on the next 10-second loop.

### 👀 Watch Your Terminal

Hit the **Upvote button** on a few questions as fast as you can.

Notice how fast the frontend updates. Now, leave the browser and look at your **terminal console**. Every 10 seconds, you should see the background worker quietly batching your clicks and logging the sync:

```bash
> `Synced total 30 votes for q_1773586009440`
> `Synced total 25 votes for q_1773586019690`
```

By doing this, you've successfully prevented "Database Locking" and built a highly scalable endpoint!

### Cleanup

Press `Ctrl+C` to stop the application. To clean up resources, run:

```bash
docker-compose down
```

---

## References

The following resources were used as references for the guide:

* [Redis Official Documentation](https://redis.io/docs)

* [Redis Quick Guides](https://redis.io/tutorials/howtos/quick-start/)
* [Getting Started with Node.js and Redis](https://redis.io/tutorials/develop/node/gettingstarted/)

## Further Reading

A few good references if you want to explore Redis beyond this guide:

* [Redis Quick Start (official docs)](https://redis.io/tutorials/howtos/quick-start/)

* [Redis Tutorial with Cheat Sheet and Quick Start Guide](https://www.dragonflydb.io/guides/beginners-redis-tutorial)

* [Redis Persistence (RDB & AOF) (official docs)](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)

* [Real-time Collaborative Editor with Redis](https://dev.to/depapp/codesync-real-time-collaborative-code-editor-powered-by-redis-15h5)

* [Redis Persistence Demystified](https://oldblog.antirez.com/post/redis-persistence-demystified.html)

## AI Declaration

Some parts of this guide were structured, formatted, and refined with the assistance of `Gemini 3.1 Pro` . The model was used to draft technical explanations and generate code snippets. All code was reviewed and tested to ensure accuracy and functionality.
