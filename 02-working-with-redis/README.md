# Getting started with Redis

In this workshop we will learn the basics of working with Redis using **film data** — the same dataset used in the PostgreSQL workshop. We will be using Docker for initialising Redis in a container.

## Table of Contents

- [What you will learn](#what-you-will-learn)
- [Prerequisites](#prerequisites)
- [Connecting to the Redis environment](#connecting-to-the-redis-environment)
- [String Data Structure](#string-data-structure)
- [List data structures](#list-data-structures)
- [Set data structures](#set-data-structures)
- [Sorted Set data structures](#sorted-set-data-structures)
- [Hash structures](#hash-structures)
- [Redis Benchmark](#redis-benchmark)
- [Working with Redis from Python](#working-with-redis-from-python)
- [Using MCP with Redis](#using-mcp-with-redis)

## What you will learn

- How to connect to Redis using the command-line utility, Redis Commander, and Redis Insight
- How to work with String data structures (GET, SET, increment/decrement, expiration/TTL)
- How to work with List data structures (watchlists, queues)
- How to work with Set data structures (genre tagging, set operations)
- How to work with Sorted Set data structures (movie rankings by rating)
- How to work with Hash data structures (movie objects)
- How to benchmark Redis performance with the built-in benchmark tool
- How to interact with Redis using the Python API
- How to use Redis MCP server to talk to the data

## Prerequisites

- The **Data Platform** described [here](../01-environment) is running and accessible

## Connecting to the Redis environment

For connecting to Redis, we can either use the Redis Command Line Utility or the browser-based Redis Commander.

### Using the Redis Command Line utility

Open another terminal window and enter the following command to start Redis CLI in another docker container:

```
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-cli -h redis-1 -p 6379
```

The Redis CLI should start and the following command prompt should appear (whereas the IP-Address can differ). 

```
redis 07:26:13.65 INFO  ==>
redis 07:26:13.65 INFO  ==> Welcome to the Bitnami redis container
redis 07:26:13.66 INFO  ==> Subscribe to project updates by watching https://github.com/bitnami/containers
redis 07:26:13.66 INFO  ==> NOTICE: Starting August 28th, 2025, only a limited subset of images/charts will remain available for free. Backup will be available for some time at the 'Bitnami Legacy' repository. More info at https://github.com/bitnami/containers/issues/83267
redis 07:26:13.66 INFO  ==>

redis-1:6379>
```

> **What you should see:** The `redis-1:6379>` prompt confirms the CLI has connected successfully to the Redis instance.

Redis is configured so that it requires authentication. You can use the `default` user, so only the password has to be passed with the `AUTH` command

```
AUTH default abc123!
```

> **What you should see:** Redis replies `OK`, confirming the password was accepted and the session is authenticated.

Enter help to see the version of Redis installed.

```
redis:6379> help
redis-cli 8.2.1
To get help about Redis commands type:
      "help @<group>" to get a list of commands in <group>
      "help <command>" for help on <command>
      "help <tab>" to get a list of possible help topics
      "quit" to exit

To set redis-cli preferences:
      ":set hints" enable online hints
      ":set nohints" disable online hints
Set your preferences in ~/.redisclirc
```

> **What you should see:** The redis-cli version number (e.g. `8.2.1`) and the help text listing available command groups.

### Using Redis Commander

In a web browser window, navigate to <http://dataplatform:28119>. You should see an image similar to the one shown below

![Redis Commander](./images/redis-commander-home.png)

> **What you should see:** The browser GUI showing the current keys stored in Redis, with a tree view on the left and a key inspector on the right.

### Using Redis Insight

In a web browser window, navigate to <http://dataplatform:28174>. Confirm the **EULA and Privacy settings** pop-up window and click **Submit** and you should see an image similar to the one shown below

![Redis Commander](./images/redis-insight-1.png)

> **What you should see:** The Redis Insight welcome screen with an option to add a new Redis database connection.

Click on **+ Add Redis database** and `redis://default@redis-1:6379` into the **Connection URL** field. Click on **Test Connection** to check the connection. If successfull, click on **Add database**.

Click on the new connection **redis-1:6379** to connect to the redis instance.

### Using DbGate

**DbGate** is cross-platform database manager. It's designed to be simple to use and effective, when working with more databases simultaneously. 

In a browser window navigate to <http://dataplatform:28120/> and login in as user `dbgate` and password `abc123!`.

On the **New Connection** page (click on the **+ Add new connection** under **CONNECTIONS** if it is not visible) select `Redis` for the **Connection type** and enter the following values:

 * **Server**: `redis-1` 
 * **Port**: `6379`
 * **User**: `default`
 * **Password**: `abc123!`

![Alt Image Text](./images/dbgate.png "DBGate Web GUI")

Click **Test** to check that connection settings are valid and then click **Connect**. 

> **What you should see:** The DbGate web UI with the Redis connection listed under CONNECTIONS. If you expand it you will see the different Redis databases, with `db0` being the default database.

## String Data Structure

Enter the commands described in the following sections at the prompt. This can be done either using the Redis CLI or using the Redis Commander (you can always use the GUIs to check the results during the workshop). 

```
HELP @STRING
```

### Working with keys

Redis is what is called a key-value store, often referred to as a NoSQL database. The essence of a key-value store is the ability to store some data, called a value, inside a key. This data can later be retrieved only if we know the exact key used to store it.

A common naming convention is to use colons as separators in key names to create a logical namespace hierarchy. For example, `movie:0110912:title` means: entity type `movie`, ID `0110912`, attribute `title`.

We can use the command `SET` to store the title of a movie:

```
SET movie:0110912:title "Pulp Fiction"
```

Redis will store our data permanently, so we can later ask _What is the title of movie 0110912_?

```
GET movie:0110912:title
```

and Redis will reply with `"Pulp Fiction"`.

> **What you should see:** Redis returns `"Pulp Fiction"`, confirming the value was stored and retrieved successfully.

Let's also store the year and rating:

```
SET movie:0110912:year "1994"
SET movie:0110912:rating "8.9"
```

We can always also just check if a key exists (from now on we show also the CLI prompt `redis:6379>` to better distinguish from the result)

```
redis:6379> EXISTS movie:0110912:title
(integer) 1
```

> **What you should see:** `(integer) 1` — a return value of 1 means the key exists in Redis.

`KEYS` searches all keys in the current Redis database that match a given glob-style pattern. The `*` wildcard matches any sequence of characters.

```
redis:6379> KEYS movie:0110912:*
1) "movie:0110912:title"
2) "movie:0110912:year"
3) "movie:0110912:rating"
```

> **What you should see:** The three keys stored for movie `0110912`, matching the wildcard pattern.

**One important caveat:** `KEYS` scans the entire keyspace in one blocking pass, so it should never be used in production against a large dataset — it will freeze Redis for every other client while it runs. The safe alternative is `SCAN`, which iterates incrementally. For workshop exploration with a small number of keys, `KEYS` is fine.

### Get and Set operations

Other common operations provided by key-value stores are `DEL` to delete a given key and associated value, `SETNX` (Set if Not eXists) that sets a key only if it does not already exist, and `INCR` to atomically increment a number stored at a given key.

Let's store the title of The Matrix:

```
redis:6379> SET movie:0133093:title "The Matrix"
OK
redis:6379> GET movie:0133093:title
"The Matrix"
```

Let's try to overwrite the Pulp Fiction title using `SETNX`. Since the key already exists, the command should have no effect:

```
redis:6379> SETNX movie:0110912:title "Something Else"
(integer) 0
redis:6379> GET movie:0110912:title
"Pulp Fiction"
```

> **What you should see:** `(integer) 0` — the key was NOT overwritten because it already existed; `GET` still returns `"Pulp Fiction"`.
>
> **What just happened?** SETNX is "Set if Not eXists" — it only writes when the key is absent, making it safe for distributed locks where only one client should be able to claim the key.

If we use `SETNX` on a key which does not yet exist, we get a different answer:

```
redis:6379> SETNX movie:0133093:year "1999"
(integer) 1
```

Let's use `MSET` to set multiple movie titles at once:

```
redis:6379> MSET movie:0111161:title "The Shawshank Redemption" movie:0068646:title "The Godfather" movie:0468569:title "The Dark Knight"
OK
```

and `MGET` to retrieve multiple values in a single round trip:

```
redis:6379> MGET movie:0111161:title movie:0068646:title movie:0468569:title
1) "The Shawshank Redemption"
2) "The Godfather"
3) "The Dark Knight"
```

> **What you should see:** All three movie titles returned in one response.
>
> **What just happened?** MGET is atomic — all values are fetched in a single server round-trip, unlike three separate GET calls which would each incur network latency.

### Increment and Decrement operation

Now let's model a movie **view counter** — a number that tracks how many times a movie has been watched on our platform.

First we initialise the counter, then use `INCR` to increment it atomically:

```
redis:6379> INCR movie:0110912:views
(integer) 1
```

> **What you should see:** `(integer) 1` — the incremented value returned immediately.

> **What just happened?** Redis auto-initialises non-existing numeric keys to 0 before applying the operation, so INCR on a missing key always yields 1.
>
> **What just happened?** INCR is an atomic operation — Redis increments and returns the new value in a single step, preventing race conditions that would occur if you read, modify, and write separately.

We can also increment by a specific amount using `INCRBY`:

```
redis:6379> INCRBY movie:0110912:views 1000
(integer) 1001
```

And decrement with `DECR` and `DECRBY`:

```
redis:6379> DECR movie:0110912:views
(integer) 1000
redis:6379> DECRBY movie:0110912:views 500
(integer) 500
```

> **What you should see:** `(integer) 500` — the value after decrementing by 500.

Now let's delete the counter and see what happens if we use `INCR` on a non-existing key:

```
redis:6379> DEL movie:0110912:views
(integer) 1
redis:6379> EXISTS movie:0110912:views
(integer) 0
redis:6379> INCR movie:0110912:views
(integer) 1
```

> **What you should see:** `(integer) 1`.

----
**Note:** There is something special about `INCR`. Why do we provide such an operation if we can do it ourselves with a bit of code? After all it is as simple as:

```
x = GET movie:0110912:views
x = x + 1
SET movie:0110912:views x
```

The problem is that doing the increment this way will only work as long as there is a single client using the key. If two clients increment the counter simultaneously, they both read the same value, both increment it, and both write the same result — effectively losing one increment. Calling `INCR` in Redis prevents this because it is an atomic operation.

----

### Expiration and Time to Live

Redis can be told that a key should only exist for a certain length of time. This is useful for "featured movie" banners, promotional offers, or any data that should expire automatically.

First let's set a key marking the currently featured movie:

```
redis:6379> SET featured:movie "Pulp Fiction"
OK
```

and then set it to expire after 2 minutes (120 seconds) using the `EXPIRE` command:

```
redis:6379> EXPIRE featured:movie 120
(integer) 1
```

Check how long the key will exist for with the `TTL` command. It returns the number of seconds until deletion:

```
redis:6379> TTL featured:movie
(integer) 96
```

> **What you should see:** A positive integer showing the remaining seconds until the key expires (the exact number will vary depending on how quickly you ran the command).

After the TTL reaches zero:

```
redis:6379> TTL featured:movie
(integer) -2
```

> **What you should see:** `(integer) -2` — this means the key has expired and been deleted.
>
> **What just happened?** Redis automatically deleted the key when its TTL reached zero — no manual cleanup needed.

You can create a key with an expiration time in one step using the `EX` option with `SET`:

```
redis:6379> SET featured:movie "The Matrix" EX 120
OK
redis:6379> TTL featured:movie
(integer) 119
```

Now use `SET` to update the featured movie without an expiry option:

```
redis:6379> SET featured:movie "Inception"
OK
redis:6379> TTL featured:movie
(integer) -1
```

> **What you should see:** `(integer) -1` — the key exists but has no expiry set.
>
> **What just happened?** Overwriting a key with `SET` (without an `EX` option) clears any previously set TTL, making the key persistent again.

Check the full list of [String commands](https://redis.io/commands#string) for more information.

## List data structures

Redis also supports several more complex data structures. The first one we'll look at is a list. A list is a series of ordered values. Some of the important commands for interacting with lists are `RPUSH`, `LPUSH`, `LLEN`, `LRANGE`, `LPOP`, and `RPOP`. You can immediately begin working with a key as a list, as long as it doesn't already exist as a different type.

We will model a **watchlist** — an ordered queue of movies a user wants to watch next.

`RPUSH` puts the new value at the end of the list.

Let's build up our watchlist by appending movies to the end:

```
redis:6379> RPUSH watchlist "Pulp Fiction"
(integer) 1
redis:6379> RPUSH watchlist "The Matrix"
(integer) 2
redis:6379> RPUSH watchlist "Inception"
(integer) 3
```

> **What you should see:** `(integer) 3` — the new length of the list after appending the third item.

Let's see what is currently in the `watchlist`. Can we use the `GET` command?

```
redis:6379> GET watchlist
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

> **What you should see:** A `WRONGTYPE` error message.
>
> **What just happened?** Redis enforces type safety per key — a key created as a **List** cannot be read with **String** commands. Each data type has its own command set.

We need to use `LRANGE` instead:

```
redis:6379> LRANGE watchlist 0 -1
1) "Pulp Fiction"
2) "The Matrix"
3) "Inception"
```

> **What you should see:** All items in insertion order — `"Pulp Fiction"` first, then `"The Matrix"`, then `"Inception"`.

`LPUSH` puts the new value at the start of the list — useful for jumping a film to the top of the queue:

```
redis:6379> LPUSH watchlist "The Dark Knight"
(integer) 4
redis:6379> LRANGE watchlist 0 -1
1) "The Dark Knight"
2) "Pulp Fiction"
3) "The Matrix"
4) "Inception"
```

`LRANGE` gives a subset of the list. It takes the index of the first element you want to retrieve as its first parameter and the index of the last element you want to retrieve as its second parameter. A value of `-1` for the second parameter means to retrieve elements until the end of the list.

```
redis:6379> LRANGE watchlist 0 1
1) "The Dark Knight"
2) "Pulp Fiction"
```

```
redis:6379> LRANGE watchlist 1 3
1) "Pulp Fiction"
2) "The Matrix"
3) "Inception"
```

`LLEN` returns the current length of the watchlist:

```
redis:6379> LLEN watchlist
(integer) 4
```

`LPOP` removes the first movie from the watchlist and returns it — the next movie to watch:

```
redis:6379> LPOP watchlist
"The Dark Knight"
```

`RPOP` removes the last movie and returns it — dropping it from the end of the queue:

```
redis:6379> RPOP watchlist
"Inception"
```

The watchlist now has two movies remaining:

```
redis:6379> LLEN watchlist
(integer) 2
redis:6379> LRANGE watchlist 0 -1
1) "Pulp Fiction"
2) "The Matrix"
```

Check the full list of [List commands](https://redis.io/commands#list) for more information.

## Set data structures

The next data structure that we'll look at is the set.

A set is similar to a list, except it does not have a specific order and each element may only appear once. Sets are ideal for **genre tagging** — a genre set contains all the movies that belong to that genre, with automatic deduplication and fast membership tests.

`SADD` adds the given value to the set.

Let's build sets of movie titles for each genre:

```
redis:6379> SADD genre:action "The Dark Knight"
(integer) 1
redis:6379> SADD genre:action "Avengers: Endgame"
(integer) 1
redis:6379> SADD genre:action "The Matrix"
(integer) 1
redis:6379> SADD genre:action "Inception"
(integer) 1
```

```
redis:6379> SADD genre:sci-fi "The Matrix"
(integer) 1
redis:6379> SADD genre:sci-fi "Inception"
(integer) 1
redis:6379> SADD genre:sci-fi "Interstellar"
(integer) 1
redis:6379> SADD genre:sci-fi "Avengers: Endgame"
(integer) 1
```

`SMEMBERS` returns all the members of a set:

```
redis:6379> SMEMBERS genre:action
1) "The Dark Knight"
2) "Avengers: Endgame"
3) "The Matrix"
4) "Inception"
```

> **What you should see:** The four movies in the action genre in arbitrary order — sets are unordered, so the output sequence may differ from what you see here.

`SREM` removes a movie from a genre:

```
redis:6379> SREM genre:action "Avengers: Endgame"
(integer) 1
redis:6379> SMEMBERS genre:action
1) "The Dark Knight"
2) "The Matrix"
3) "Inception"
```

`SISMEMBER` tests if a movie belongs to a genre:

```
redis:6379> SISMEMBER genre:action "The Dark Knight"
(integer) 1
redis:6379> SISMEMBER genre:action "Pulp Fiction"
(integer) 0
```

> **What you should see:** `(integer) 1` for The Dark Knight (it is a member) and `(integer) 0` for Pulp Fiction — 0 means not a member of the set.

`SUNION` combines two or more sets and returns all elements — every movie tagged as Action OR Sci-Fi:

```
redis:6379> SUNION genre:action genre:sci-fi
1) "The Dark Knight"
2) "The Matrix"
3) "Inception"
4) "Interstellar"
5) "Avengers: Endgame"
```

> **What you should see:** The combined unique elements from both sets (order may vary).
>
> **What just happened?** `SUNION` merges two sets and deduplicates — equivalent to a set union in mathematics. Any element present in either set appears exactly once in the result. The Matrix and Inception appear only once even though they are in both sets.

`SUNIONSTORE` combines two or more sets and stores the result into a new set:

```
redis:6379> SUNIONSTORE genre:action-or-sci-fi genre:action genre:sci-fi
(integer) 5
redis:6379> SMEMBERS genre:action-or-sci-fi
```

`SINTER` intersects two or more sets — movies tagged as both Action AND Sci-Fi:

```
redis:6379> SINTER genre:action genre:sci-fi
1) "The Matrix"
2) "Inception"
```

> **What you should see:** Only the elements present in both sets — `"The Matrix"` and `"Inception"` (order may vary).
>
> **What just happened?** `SINTER` returns the intersection — only members that exist in every specified set. Movies unique to one genre are excluded.

`SDIFF` returns movies in the first set but not in the second — movies tagged as Action but NOT Sci-Fi:

```
redis:6379> SDIFF genre:action genre:sci-fi
1) "The Dark Knight"
```

Check the full list of [Set commands](https://redis.io/commands#set) for more information.

## Sorted Set data structures

Sets are a very handy data type, but as they are unsorted they don't work well for ranking. This is why Redis 1.2 introduced Sorted Sets.

A sorted set is similar to a regular set, but now each value has an associated score. This score is used to sort the elements in the set. Movie ratings are a natural fit for the score — we can build a **real-time leaderboard** of the top-rated films.

`ZADD` adds one or more members to a sorted set, or updates the score if it already exists.

```
redis:6379> ZADD top:movies 9.2 "The Shawshank Redemption"
(integer) 1
redis:6379> ZADD top:movies 9.2 "The Godfather"
(integer) 1
redis:6379> ZADD top:movies 9.0 "The Godfather: Part II"
(integer) 1
redis:6379> ZADD top:movies 9.0 "The Dark Knight"
(integer) 1
redis:6379> ZADD top:movies 8.9 "12 Angry Men"
(integer) 1
redis:6379> ZADD top:movies 8.9 "Pulp Fiction"
(integer) 1
redis:6379> ZADD top:movies 8.9 "Schindler's List"
(integer) 1
redis:6379> ZADD top:movies 8.7 "The Matrix"
(integer) 1
redis:6379> ZADD top:movies 8.7 "Inception"
(integer) 1
redis:6379> ZADD top:movies 8.7 "Forrest Gump"
(integer) 1
```

`ZREVRANGE` returns a range of members in descending score order (highest rated first):

```
redis:6379> ZREVRANGE top:movies 0 2
1) "The Shawshank Redemption"
2) "The Godfather"
3) "The Dark Knight"
```

> **What you should see:** The three highest-rated movies. When two movies share the same score, Redis orders them lexicographically by member name, which is why The Godfather comes after The Shawshank Redemption (both rated 9.2) and The Dark Knight appears before The Godfather: Part II (both rated 9.0).

`ZRANGE` returns members in ascending score order (lowest rated first):

```
redis:6379> ZRANGE top:movies 0 2
1) "Forrest Gump"
2) "Inception"
3) "The Matrix"
```

`ZREVRANGE` with `WITHSCORES` — ratings alongside titles:

```
redis:6379> ZREVRANGE top:movies 0 4 WITHSCORES
 1) "The Shawshank Redemption"
 2) "9.2"
 3) "The Godfather"
 4) "9.2"
 5) "The Dark Knight"
 6) "9"
 7) "The Godfather: Part II"
 8) "9"
 9) "12 Angry Men"
10) "8.9"
```

> **What you should see:** Movie titles and their IMDb ratings interleaved, ordered highest score first.

`ZSCORE` — look up the rating of a specific movie:

```
redis:6379> ZSCORE top:movies "Pulp Fiction"
"8.9"
```

`ZRANK` — the rank of a movie, ascending (0 = lowest rated):

```
redis:6379> ZRANK top:movies "The Matrix"
(integer) 2
```

`ZREVRANK` — the rank of a movie, descending (0 = highest rated):

```
redis:6379> ZREVRANK top:movies "The Shawshank Redemption"
(integer) 0
redis:6379> ZREVRANK top:movies "The Matrix"
(integer) 7
```

> **What you should see:** The Shawshank Redemption is ranked 0 (the top-rated), while The Matrix is ranked 7th from the top.

Check the full list of [Sorted Set commands](https://redis.io/commands#sorted_set) for more information.

## Hash structures

Simple strings, sets and sorted sets already get a lot done but there is one more data type Redis can handle: Hashes.

Hashes are maps between string fields and string values, so they are the perfect data type to represent objects — for example, a **movie** with fields like title, year, runtime, rating, and vote count.

`HSET` sets one or more fields in the hash stored at the given key:

```
redis:6379> HSET movie:0110912 title "Pulp Fiction" year 1994 runtime 154 rating 8.9 votes 2084331
(integer) 5
```

To get back all the saved fields use `HGETALL`:

```
redis:6379> HGETALL movie:0110912
 1) "title"
 2) "Pulp Fiction"
 3) "year"
 4) "1994"
 5) "runtime"
 6) "154"
 7) "rating"
 8) "8.9"
 9) "votes"
10) "2084331"
```

> **What you should see:** All field-value pairs stored in the hash for `movie:0110912`, returned as an alternating list of field names and values.

Let's also store The Matrix:

```
redis:6379> HSET movie:0133093 title "The Matrix" year 1999 runtime 136 rating 8.7 votes 1496538
(integer) 5
```

`HGET` retrieves a single field:

```
redis:6379> HGET movie:0110912 title
"Pulp Fiction"
redis:6379> HGET movie:0110912 rating
"8.9"
```

`HMSET` is an older variant that sets multiple fields at once (in modern Redis, `HSET` already accepts multiple field-value pairs):

```
redis:6379> HMSET movie:0111161 title "The Shawshank Redemption" year 1994 rating 9.2
OK
redis:6379> HGETALL movie:0111161
1) "title"
2) "The Shawshank Redemption"
3) "year"
4) "1994"
5) "rating"
6) "9.2"
```

Numerical values in hash fields are handled exactly the same as in simple strings and there are operations to increment this value in an atomic way. Let's simulate a new vote being cast for Pulp Fiction:

```
redis:6379> HGET movie:0110912 votes
"2084331"
redis:6379> HINCRBY movie:0110912 votes 1
(integer) 2084332
redis:6379> HINCRBY movie:0110912 votes 1000
(integer) 2085332
```

> **What you should see:** The vote count incrementing atomically.
>
> **What just happened?** HINCRBY atomically increments a numeric field inside a hash — the same atomicity guarantee as INCR on strings, preventing race conditions in concurrent access scenarios.

`HDEL` removes a specific field from the hash:

```
redis:6379> HDEL movie:0110912 votes
(integer) 1
redis:6379> HGETALL movie:0110912
1) "title"
2) "Pulp Fiction"
3) "year"
4) "1994"
5) "runtime"
6) "154"
7) "rating"
8) "8.9"
```

Check the full list of [Hash commands](https://redis.io/commands#hash) for more information.

## Redis Benchmark

Redis ships with a built-in benchmarking tool called `redis-benchmark`. It simulates a number of clients sending commands in parallel and reports the throughput (requests/second) for each command type. This is useful for understanding the performance characteristics of your Redis deployment and for comparing the effect of different configurations.

### Running a basic benchmark

Run the benchmark tool from a temporary Docker container on the same Docker network as Redis:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -q
```

The key flags used here are:

| Flag | Description |
|------|-------------|
| `-h redis-1` | Hostname of the Redis server |
| `-p 6379` | Port of the Redis server |
| `-n 100000` | Total number of requests to send per test |
| `-q` | Quiet mode — only prints the summary line (ops/sec) per command |

You should see output similar to the following (exact numbers will vary depending on your hardware):

```
PING_INLINE: 60132.29 requests per second, p50=0.439 msec
PING_MBULK: 62227.75 requests per second, p50=0.399 msec
SET: 65445.03 requests per second, p50=0.391 msec
GET: 50454.09 requests per second, p50=0.495 msec
INCR: 64683.05 requests per second, p50=0.391 msec
LPUSH: 60679.61 requests per second, p50=0.423 msec
RPUSH: 54644.81 requests per second, p50=0.455 msec
LPOP: 57803.47 requests per second, p50=0.543 msec
RPOP: 60386.47 requests per second, p50=0.415 msec
SADD: 60386.47 requests per second, p50=0.399 msec
HSET: 57736.72 requests per second, p50=0.463 msec
SPOP: 60096.15 requests per second, p50=0.407 msec
ZADD: 59808.61 requests per second, p50=0.407 msec
ZPOPMIN: 59772.86 requests per second, p50=0.415 msec
LPUSH (needed to benchmark LRANGE): 54704.60 requests per second, p50=0.527 msec
LRANGE_100 (first 100 elements): 38138.82 requests per second, p50=0.631 msec
LRANGE_300 (first 300 elements): 17307.03 requests per second, p50=1.335 msec
LRANGE_500 (first 500 elements): 10895.62 requests per second, p50=2.119 msec
LRANGE_600 (first 600 elements): 9694.62 requests per second, p50=2.359 msec
MSET (10 keys): 47415.84 requests per second, p50=0.663 msec
XADD: 56338.03 requests per second, p50=0.479 msec
```

> **What you should see:** One line per command showing the throughput in requests per second and the p50 (median) latency. Higher requests/second means better throughput; lower latency is better.

### Running a more verbose benchmark

Remove the `-q` flag to get detailed per-percentile latency histograms for each command:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000
```

You will see output like this for each command:

```
====== SET ======
  100000 requests completed in 1.84 seconds
  50 parallel clients
  3 bytes payload
  keep alive: 1
  host configuration "save":
  host configuration "appendonly": yes
  multi-thread: no

Latency by percentile distribution:
0.000% <= 0.127 milliseconds (cumulative count 1)
50.000% <= 0.447 milliseconds (cumulative count 50276)
75.000% <= 0.543 milliseconds (cumulative count 76721)
87.500% <= 0.615 milliseconds (cumulative count 87647)
93.750% <= 0.879 milliseconds (cumulative count 93757)
96.875% <= 1.303 milliseconds (cumulative count 96885)
98.438% <= 1.679 milliseconds (cumulative count 98440)
99.219% <= 2.583 milliseconds (cumulative count 99220)
99.609% <= 3.919 milliseconds (cumulative count 99611)
99.805% <= 6.055 milliseconds (cumulative count 99805)
99.902% <= 10.063 milliseconds (cumulative count 99903)
99.951% <= 12.943 milliseconds (cumulative count 99952)
99.976% <= 17.567 milliseconds (cumulative count 99976)
99.988% <= 17.951 milliseconds (cumulative count 99988)
99.994% <= 18.095 milliseconds (cumulative count 99994)
99.997% <= 18.175 milliseconds (cumulative count 99997)
99.998% <= 18.255 milliseconds (cumulative count 99999)
99.999% <= 18.319 milliseconds (cumulative count 100000)
100.000% <= 18.319 milliseconds (cumulative count 100000)

Cumulative distribution of latencies:
0.000% <= 0.103 milliseconds (cumulative count 0)
0.027% <= 0.207 milliseconds (cumulative count 27)
0.759% <= 0.303 milliseconds (cumulative count 759)
36.998% <= 0.407 milliseconds (cumulative count 36998)
65.700% <= 0.503 milliseconds (cumulative count 65700)
87.143% <= 0.607 milliseconds (cumulative count 87143)
90.552% <= 0.703 milliseconds (cumulative count 90552)
92.638% <= 0.807 milliseconds (cumulative count 92638)
94.056% <= 0.903 milliseconds (cumulative count 94056)
95.068% <= 1.007 milliseconds (cumulative count 95068)
95.782% <= 1.103 milliseconds (cumulative count 95782)
96.397% <= 1.207 milliseconds (cumulative count 96397)
96.885% <= 1.303 milliseconds (cumulative count 96885)
97.372% <= 1.407 milliseconds (cumulative count 97372)
97.806% <= 1.503 milliseconds (cumulative count 97806)
98.239% <= 1.607 milliseconds (cumulative count 98239)
98.488% <= 1.703 milliseconds (cumulative count 98488)
98.672% <= 1.807 milliseconds (cumulative count 98672)
98.802% <= 1.903 milliseconds (cumulative count 98802)
98.901% <= 2.007 milliseconds (cumulative count 98901)
98.999% <= 2.103 milliseconds (cumulative count 98999)
99.351% <= 3.103 milliseconds (cumulative count 99351)
99.659% <= 4.103 milliseconds (cumulative count 99659)
99.721% <= 5.103 milliseconds (cumulative count 99721)
99.812% <= 6.103 milliseconds (cumulative count 99812)
99.898% <= 7.103 milliseconds (cumulative count 99898)
99.900% <= 8.103 milliseconds (cumulative count 99900)
99.906% <= 10.103 milliseconds (cumulative count 99906)
99.948% <= 11.103 milliseconds (cumulative count 99948)
99.950% <= 12.103 milliseconds (cumulative count 99950)
99.956% <= 13.103 milliseconds (cumulative count 99956)
99.959% <= 14.103 milliseconds (cumulative count 99959)
99.960% <= 17.103 milliseconds (cumulative count 99960)
99.994% <= 18.111 milliseconds (cumulative count 99994)
100.000% <= 19.103 milliseconds (cumulative count 100000)

Summary:
  throughput summary: 54436.58 requests per second
  latency summary (msec):
          avg       min       p50       p95       p99       max
        0.543     0.120     0.447     1.007     2.111    18.319
```

### Benchmarking specific commands

Use the `-t` flag to benchmark only specific commands (comma-separated):

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -t set,get,incr -q
```

```
SET: 61012.81 requests per second, p50=0.407 msec
GET: 61349.70 requests per second, p50=0.399 msec
INCR: 58072.01 requests per second, p50=0.447 msec
```

> **What you should see:** Throughput numbers only for the three specified commands.

### Effect of concurrent connections

The `-c` flag controls the number of parallel client connections (default is 50). Increasing it can reveal throughput gains from parallelism or expose contention:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -c 10 -t set,get -q
```

```
SET: 61462.82 requests per second, p50=0.095 msec
GET: 52219.32 requests per second, p50=0.111 msec
```

Now increase to 20 connections:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -c 20 -t set,get -q
```

```
SET: 58105.75 requests per second, p50=0.191 msec
GET: 65146.58 requests per second, p50=0.159 msec
```

> **What you should see:** Higher throughput at 20 connections compared to 10, but also higher latency. This demonstrates the throughput vs. latency trade-off under concurrency.

### Effect of payload size

The `-d` flag controls the size of the value in bytes used for SET/GET tests (default is 3 bytes). Larger payloads take more time to serialise and transmit:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -d 64 -t set,get -q
```

```
SET: 59594.76 requests per second, p50=0.415 msec
GET: 59844.41 requests per second, p50=0.423 msec
```

Try a larger payload of 1024 bytes (1 KB):

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -d 1024 -t set,get -q
```

```
SET: 48638.13 requests per second, p50=0.551 msec
GET: 52410.90 requests per second, p50=0.439 msec
```

> **What you should see:** A small but measurable drop in throughput as payload size increases, because Redis must read and write more bytes per operation.

### Effect of pipelining

Pipelining batches multiple commands into a single network round trip. The `-P` flag sets the number of commands per pipeline batch (default is 1, i.e. no pipelining):

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -P 8 -t set,get -q
```

```
SET: 293255.12 requests per second, p50=0.959 msec
GET: 420168.06 requests per second, p50=0.471 msec
```

> **What you should see:** A dramatic increase in throughput — often 5–10x or more — because each network round trip now carries 16 commands instead of 1.
>
> **What just happened?** Pipelining removes the per-command network round-trip overhead. This is the single biggest performance lever available at the client level and is commonly used in batch-write scenarios.

### Exporting results as CSV

Use `--csv` to produce machine-readable output suitable for loading into a spreadsheet or monitoring tool:

```bash
docker run -it --rm --network modern-database-platform-1 bitnamilegacy/redis:8.2 redis-benchmark -h redis-1 -p 6379 -n 100000 -t set,get,incr -q --csv
```

```
"test","rps","avg_latency_ms","min_latency_ms","p50_latency_ms","p95_latency_ms","p99_latency_ms","max_latency_ms"
"SET","51652.89","0.562","0.152","0.463","0.895","2.671","16.607"
"GET","49603.17","0.583","0.112","0.479","0.775","2.055","60.095"
"INCR","54794.52","0.581","0.160","0.495","0.967","1.743","21.407"
```

> **What you should see:** One CSV line per tested command with the command name and its throughput in requests per second.

## Working with Redis from Python

The `redis-py` library is the standard Python client for Redis. In this section we will connect to Redis from the **Jupyter** environment and reproduce the same operations we performed in the CLI sections above — this time using the film dataset.

Open a browser and navigate to <http://dataplatform:28888> and login it with token `abc123!`.
 
Create a new Python 3 notebook by clicking on the Python 3 (ipykernel) widget. 

![](./images/jupyter-1.png)

A new notebook will be created with a first cell available to execute commands. 

![](./images/jupyter-2.png)

Then work through the cells below in order.

### Cell 1 — Install the library

```python
import sys
!{sys.executable} -m pip install redis
```

> **What you should see:** pip output ending with `Successfully installed redis-...`.

### Cell 2 — Connect to Redis

```python
import redis

r = redis.Redis(host='redis-1', port=6379, db=0, decode_responses=True)
#r.auth('abc123!')
r.ping()
```

> **What you should see:** `True` — the client successfully reached Redis and received a `PONG` response.

`decode_responses=True` tells the client to return Python strings instead of raw bytes, which makes notebook output much easier to read.

### Cell 3 — String operations with movie data

The Python method names map directly onto the Redis commands you already know:

```python
# Store individual movie attributes
r.set('movie:0110912:title', 'Pulp Fiction')
r.set('movie:0110912:year',  '1994')
r.set('movie:0110912:rating','8.9')

print(r.get('movie:0110912:title'))    # Pulp Fiction
print(r.exists('movie:0110912:title')) # 1

# Set multiple titles at once
r.mset({
    'movie:0111161:title': 'The Shawshank Redemption',
    'movie:0068646:title': 'The Godfather',
    'movie:0468569:title': 'The Dark Knight',
})
print(r.mget('movie:0111161:title', 'movie:0068646:title', 'movie:0468569:title'))
# ['The Shawshank Redemption', 'The Godfather', 'The Dark Knight']

# SETNX — only sets the key if it does not already exist
print(r.setnx('movie:0110912:title', 'Something Else'))  # False — key exists
print(r.get('movie:0110912:title'))                       # Pulp Fiction

# Featured movie with a TTL
r.set('featured:movie', 'The Matrix', ex=120)
print(r.ttl('featured:movie'))         # ~120
```

> **What you should see:** Each `print` call returns the value described in the inline comment.

### Cell 4 — Increment and Decrement (view counter)

```python
# Initialise and increment a movie view counter
r.set('movie:0110912:views', 0)
print(r.incr('movie:0110912:views'))           # 1
print(r.incrby('movie:0110912:views', 1000))   # 1001
print(r.decr('movie:0110912:views'))           # 1000
print(r.decrby('movie:0110912:views', 500))    # 500

# INCR on a non-existing key starts from 0
r.delete('movie:0133093:views')
print(r.incr('movie:0133093:views'))           # 1
```

### Cell 5 — List operations (watchlist)

```python
r.delete('watchlist')

# Build a watchlist
r.rpush('watchlist', 'Pulp Fiction')
r.rpush('watchlist', 'The Matrix')
r.rpush('watchlist', 'Inception')
r.lpush('watchlist', 'The Dark Knight')   # jump to the front of the queue

print(r.lrange('watchlist', 0, -1))
# ['The Dark Knight', 'Pulp Fiction', 'The Matrix', 'Inception']

print(r.llen('watchlist'))                # 4

# Watch the next movie (remove from the front)
print(r.lpop('watchlist'))                # The Dark Knight

# Drop the last movie from the queue
print(r.rpop('watchlist'))                # Inception

print(r.lrange('watchlist', 0, -1))
# ['Pulp Fiction', 'The Matrix']
```

### Cell 6 — Set operations (genre tagging)

```python
r.delete('genre:action', 'genre:sci-fi', 'genre:action-or-sci-fi')

r.sadd('genre:action', 'The Dark Knight', 'Avengers: Endgame', 'The Matrix', 'Inception')
r.sadd('genre:sci-fi', 'The Matrix', 'Inception', 'Interstellar', 'Avengers: Endgame')

print(r.smembers('genre:action'))
# {'The Dark Knight', 'Avengers: Endgame', 'The Matrix', 'Inception'}  (order may vary)

# Membership test
print(r.sismember('genre:action', 'The Dark Knight'))  # True
print(r.sismember('genre:action', 'Pulp Fiction'))     # False

# Remove a movie from a genre
r.srem('genre:action', 'Avengers: Endgame')

# SUNION — all movies tagged as Action OR Sci-Fi
print(r.sunion('genre:action', 'genre:sci-fi'))
# {'The Dark Knight', 'The Matrix', 'Inception', 'Interstellar', 'Avengers: Endgame'}

# SINTER — movies tagged as both Action AND Sci-Fi
print(r.sinter('genre:action', 'genre:sci-fi'))
# {'The Matrix', 'Inception'}

# SDIFF — movies tagged as Action but NOT Sci-Fi
print(r.sdiff('genre:action', 'genre:sci-fi'))
# {'The Dark Knight'}
```

> **What you should see:** Set membership results matching the comments. The order of set elements may vary — sets are unordered.

### Cell 7 — Sorted Set operations (movie rankings)

```python
r.delete('top:movies')

ratings = {
    'The Shawshank Redemption': 9.2,
    'The Godfather':            9.2,
    "The Godfather: Part II":   9.0,
    'The Dark Knight':          9.0,
    '12 Angry Men':             8.9,
    'Pulp Fiction':             8.9,
    "Schindler's List":         8.9,
    'The Matrix':               8.7,
    'Inception':                8.7,
    'Forrest Gump':             8.7,
}
r.zadd('top:movies', ratings)

# ZREVRANGE — top 5 highest-rated (descending), with scores
print(r.zrevrange('top:movies', 0, 4, withscores=True))
# [('The Shawshank Redemption', 9.2), ('The Godfather', 9.2),
#  ('The Dark Knight', 9.0), ("The Godfather: Part II", 9.0), ("Schindler's List", 8.9)]

# ZRANGE — lowest-rated first (ascending), first 3
print(r.zrange('top:movies', 0, 2))
# ['Forrest Gump', 'Inception', 'The Matrix']

# ZSCORE — look up a specific movie's rating
print(r.zscore('top:movies', 'Pulp Fiction'))      # 8.9

# ZREVRANK — position in the leaderboard (0 = highest rated)
print(r.zrevrank('top:movies', 'The Shawshank Redemption'))  # 0
print(r.zrevrank('top:movies', 'The Matrix'))                 # 7
```

### Cell 8 — Hash operations (movie objects)

```python
r.delete('movie:0110912', 'movie:0133093', 'movie:0111161')

# Store a full movie object as a hash
r.hset('movie:0110912', mapping={
    'title':   'Pulp Fiction',
    'year':    '1994',
    'runtime': '154',
    'rating':  '8.9',
    'votes':   '2084331',
})

print(r.hgetall('movie:0110912'))
# {'title': 'Pulp Fiction', 'year': '1994', 'runtime': '154', 'rating': '8.9', 'votes': '2084331'}

r.hset('movie:0133093', mapping={
    'title':   'The Matrix',
    'year':    '1999',
    'runtime': '136',
    'rating':  '8.7',
    'votes':   '1496538',
})

# HGET — retrieve a single field
print(r.hget('movie:0110912', 'title'))    # Pulp Fiction
print(r.hget('movie:0110912', 'rating'))   # 8.9

# HINCRBY — atomically increment the vote count
print(r.hincrby('movie:0110912', 'votes', 1))       # 2084332
print(r.hincrby('movie:0110912', 'votes', 1000))    # 2085332

# HDEL — remove a field
r.hdel('movie:0110912', 'votes')
print(r.hgetall('movie:0110912'))
# {'title': 'Pulp Fiction', 'year': '1994', 'runtime': '154', 'rating': '8.9'}
```

### Cell 9 — Bulk writes with a pipeline

Just like `redis-benchmark -P` batches commands, `redis-py` has a pipeline API that sends multiple commands in a single round trip. Here we bulk-load the top-50 movie titles:

```python
top_movies = [
    ('0111161', 'The Shawshank Redemption',                              1994, 9.2),
    ('0068646', 'The Godfather',                                         1972, 9.2),
    ('0071562', 'The Godfather: Part II',                                1974, 9.0),
    ('0468569', 'The Dark Knight',                                       2008, 9.0),
    ('0050083', '12 Angry Men',                                          1957, 8.9),
    ('0110912', 'Pulp Fiction',                                          1994, 8.9),
    ('0108052', "Schindler's List",                                      1993, 8.9),
    ('0167260', 'The Lord of the Rings: The Return of the King',         2003, 8.9),
    ('0060196', 'The Good, the Bad and the Ugly',                        1966, 8.8),
    ('0137523', 'Fight Club',                                            1999, 8.8),
    ('4154796', 'Avengers: Endgame',                                     2019, 8.8),
    ('0120737', 'The Lord of the Rings: The Fellowship of the Ring',     2001, 8.8),
    ('0109830', 'Forrest Gump',                                          1994, 8.7),
    ('0080684', 'Star Wars: Episode V - The Empire Strikes Back',        1980, 8.7),
    ('1375666', 'Inception',                                             2010, 8.7),
    ('0167261', 'The Lord of the Rings: The Two Towers',                 2002, 8.7),
    ('0073486', "One Flew Over the Cuckoo's Nest",                       1975, 8.7),
    ('0099685', 'Goodfellas',                                            1990, 8.7),
    ('0133093', 'The Matrix',                                            1999, 8.7),
    ('0047478', 'Seven Samurai',                                         1954, 8.6),
    ('0114369', 'Se7en',                                                 1995, 8.6),
    ('0317248', 'City of God',                                           2002, 8.6),
    ('0076759', 'Star Wars: Episode IV - A New Hope',                    1977, 8.6),
    ('0102926', 'The Silence of the Lambs',                              1991, 8.6),
    ('0038650', "It's a Wonderful Life",                                 1946, 8.6),
    ('0118799', 'Life Is Beautiful',                                     1997, 8.6),
    ('0245429', 'Spirited Away',                                         2001, 8.5),
    ('0120815', 'Saving Private Ryan',                                   1998, 8.5),
    ('0114814', 'The Usual Suspects',                                    1995, 8.5),
    ('0110413', 'Léon: The Professional',                                1994, 8.5),
    ('0120689', 'The Green Mile',                                        1999, 8.5),
    ('0816692', 'Interstellar',                                          2014, 8.5),
    ('0054215', 'Psycho',                                                1960, 8.5),
    ('0120586', 'American History X',                                    1998, 8.5),
    ('0021749', 'City Lights',                                           1931, 8.5),
    ('0034583', 'Casablanca',                                            1942, 8.5),
    ('0064116', 'Once Upon a Time in the West',                          1968, 8.5),
    ('0253474', 'The Pianist',                                           2002, 8.5),
    ('0027977', 'Modern Times',                                          1936, 8.5),
    ('1675434', 'The Intouchables',                                      2011, 8.5),
    ('0407887', 'The Departed',                                          2006, 8.5),
    ('0088763', 'Back to the Future',                                    1985, 8.5),
    ('0103064', 'Terminator 2: Judgment Day',                            1991, 8.5),
    ('2582802', 'Whiplash',                                              2014, 8.5),
    ('0110357', 'The Lion King',                                         1994, 8.5),
    ('0047396', 'Rear Window',                                           1954, 8.5),
    ('0082971', 'Raiders of the Lost Ark',                               1981, 8.5),
    ('0172495', 'Gladiator',                                             2000, 8.5),
    ('0482571', 'The Prestige',                                          2006, 8.5),
    ('0078788', 'Apocalypse Now',                                        1979, 8.4),
]

pipe = r.pipeline()

for movie_id, title, year, rating in top_movies:
    key = f'movie:{movie_id}'
    pipe.hset(key, mapping={'title': title, 'year': str(year), 'rating': str(rating)})
    pipe.zadd('top50:movies', {title: rating})

pipe.execute()   # all commands sent in one round trip

print(f'Loaded {len(top_movies)} movies.')

# Verify: top 5 by rating
print(r.zrevrange('top50:movies', 0, 4, withscores=True))
```

> **What you should see:** `Loaded 50 movies.` followed by the top 5 highest-rated movies.
>
> **What just happened?** `pipeline()` buffers commands locally and flushes them all at once when `execute()` is called, eliminating per-command network latency — the Python equivalent of the pipelining you measured with `redis-benchmark -P`.

### Cell 10 — Cleaning up

Remove all keys created during this workshop:

```python
pattern_groups = [
    'movie:*',
    'top:movies',
    'top50:movies',
    'genre:*',
    'watchlist',
    'featured:movie',
]

keys_to_delete = []
for pattern in pattern_groups:
    keys_to_delete.extend(r.keys(pattern))

if keys_to_delete:
    r.delete(*keys_to_delete)
    print(f'Deleted {len(keys_to_delete)} keys.')
else:
    print('Nothing to delete.')
```

> **What you should see:** `Deleted N keys.` confirming all workshop keys have been removed.

## Using MCP with Redis

**Model Context Protocol (MCP)** is an open standard that lets AI assistants connect to external tools and data sources through a common interface. Instead of running Redis commands manually, you can give an AI assistant access to your Redis instance via an MCP server and interact with it in natural language.

The data platform includes two components for this:

| Component | URL | Purpose |
|-----------|-----|---------|
| **Redis MCP Server** | `http://dataplatform:28225/sse` | Exposes Redis as an MCP tool (SSE transport) |
| **MCP Inspector** | <http://dataplatform:6274> | Web UI for exploring and testing any MCP server |

The Redis MCP server is [mcp-redis](https://github.com/redis/mcp-redis) by Redis, running as version `0.5.0` and connected to `redis://redis-1:6379/0` (database 0).

### Exploring the MCP server with MCP Inspector

MCP Inspector is a browser-based tool for browsing the tools an MCP server exposes and for sending test requests.

1. Open <http://dataplatform:6274> in your browser.
2. In the **Transport** dropdown, select **SSE**.
3. In the **URL** field, enter:
   ```
   http://dataplatform:28225/sse
   ```
4. Click **Connect**.

> **What you should see:** The Inspector connects and lists the MCP server's capabilities — tools covering all major Redis data structures: strings, lists, sets, sorted sets, hashes, streams, and more.

### Browsing tools

Once connected, click the **Tools** tab. You will see the full list of tools the Redis MCP server exposes. Click any tool name to expand its input schema and description.

Key tools available (grouped by data type):

| Category | Example tools |
|----------|--------------|
| **Strings** | `set`, `get`, `mset`, `mget`, `incr`, `incrby`, `decr`, `expire`, `ttl` |
| **Lists** | `rpush`, `lpush`, `lrange`, `lpop`, `rpop`, `llen` |
| **Sets** | `sadd`, `smembers`, `srem`, `sismember`, `sunion`, `sinter` |
| **Sorted Sets** | `zadd`, `zrange`, `zrevrange`, `zrank`, `zscore` |
| **Hashes** | `hset`, `hget`, `hgetall`, `hmset`, `hincrby`, `hdel` |
| **Keys** | `keys`, `exists`, `del`, `type`, `scan` |

### Running a command from MCP Inspector

1. Click the **Tools** tab and select **hset**.
2. In the **Arguments** panel, enter:
   ```json
   {
     "key": "movie:0110912",
     "field": "title",
     "value": "Pulp Fiction"
   }
   ```
3. Click **Run Tool**.

> **What you should see:** A JSON response confirming the field was set (`1`).

Now read the full movie hash back:

1. Select the **hgetall** tool.
2. Enter:
   ```json
   { "key": "movie:0110912" }
   ```
3. Click **Run Tool**.

> **What you should see:** The hash fields for `movie:0110912` returned as a JSON object.

### Exploring stored keys

To see what movie keys are currently in Redis, select the **keys** tool and enter:

```json
{ "pattern": "movie:*" }
```

> **What you should see:** A list of all movie keys currently stored in the Redis instance.

### Connecting an AI assistant to the MCP server

Any MCP-compatible AI client (Claude Desktop, Cursor, VS Code with an MCP extension) can connect to the Redis MCP server. Configure the client with:

- **Transport**: SSE
- **URL**: `http://dataplatform:28225/sse`

Once connected, the assistant can answer questions and perform actions like:

- *"What is the rating of Pulp Fiction?"*
- *"Add 'The Shawshank Redemption' to the Action genre set."*
- *"Which movies are tagged as both Action and Sci-Fi?"*
- *"Show me the top 5 movies by rating from the top50:movies sorted set."*
- *"Increment the view counter for movie 0110912 by 1."*
- *"Store the movie 'Gladiator' with year 2000 and rating 8.5 in a hash."*

The assistant translates these into Redis commands that execute against the live Redis instance, and returns the results in plain language.

> **What just happened?** The MCP server acts as a bridge between the AI assistant and Redis. The assistant sees a typed tool interface — each tool has a name, description, and input schema — rather than a raw socket connection. This means the assistant knows which commands are available, what arguments they accept, and can validate inputs before sending them, making AI-assisted data operations safer and more predictable.
