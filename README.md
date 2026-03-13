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

