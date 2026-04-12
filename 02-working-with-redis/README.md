# Getting started with Redis

In this workshop we will learn the basics of working with Redis. We will be using Docker for initialising Redis in a container. 

## What you will learn

- How to connect to Redis using the command-line utility, Redis Commander, and Redis Insight
- How to work with String data structures (GET, SET, increment/decrement, expiration/TTL)
- How to work with Set data structures
- How to work with Sorted Set data structures
- How to work with Hash data structures
- How to benchmark Redis performance with the built-in benchmark tool
- How to interact with Redis using the Python API

## Prerequisites

- The **Data Platform** described [here](../01-environment) is running and accessible

## Connecting to the Redis environment

For connecting to Redis, we can either use the Redis Command Line Utility or the browser-based Redis Commander.

### Using the Redis Command Line utility

Open another terminal window and enter the following command to start Redis CLI in another docker container:

```
docker run -it --rm --network nosql-platform bitnami/redis redis-cli -h redis-1 -p 6379
```

The Redis CLI should start and the following command prompt should appear (whereas the IP-Address can differ). 

```
redis 21:04:06.73
redis 21:04:06.74 Welcome to the Bitnami redis container
redis 21:04:06.74 Subscribe to project updates by watching https://github.com/bitnami/bitnami-docker-redis
redis 21:04:06.74 Submit issues and feature requests at https://github.com/bitnami/bitnami-docker-redis/issues
redis 21:04:06.74

redis-1:6379> 
```

Redis is configured so that it requires authentication. You can use the `default` user, so only the password has to be passed with the `AUTH` command

```
AUTH abc123!
```

Enter help to see the version of Redis installed.

```
redis:6379> help
redis-cli 6.0.9
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

### Using Redis Commander

In a web browser window, navigate to <http://dataplatform:28119>. You should see an image similar to the one shown below

![Redis Commander](./images/redis-commander-home.png)

### Using Redis Insight

In a web browser window, navigate to <http://dataplatform:28174>. You should see an image similar to the one shown below

![Redis Commander](./images/redis-insight-1.png)

Click on **+ Add Redis database** and `redis://default@redis-1:6379` into the **Connection URL** field. Click on **Test Connection** to check if the connection is possible. If successfull, click on **Add Database**.

Click on the new connection `redis-1:6379` to connect to the redis instance.


## String Data Structure

Enter the commands described in the following sections at the prompt. This can be done either using the Redis CLI or using the Redis Commander. 

```
HELP @STRING
```

###	Working with keys

Redis is what is called a key-value store, often referred to as a NoSQL database. The essence of a key-value store is the ability to store some data, called a value, inside a key. This data can later be retrieved only if we know the exact key used to store it. 

We can use the command `SET` to store the value “redis-server” at key “server:name”:

```
SET server:name "redis-server"
```

Redis will store our data permanently, so we can later ask _What is the value stored at key server:name_?

```
GET server:name 
```

and Redis will reply with “redis-server”.

```
EXISTS server:name
(integer) 1
```

```
KEYS server*
1) "server:name"
```

```
KEYS *
1) "server:name"
```

### Get and Set operations

Other common operations provided by key-value stores are DEL to delete a given key and associated value, SET-if-not-exists (called SETNX on Redis) that sets a key only if it does not already exist, and INCR to atomically increment a number stored at a given key. So let's see some of these commands in action:

Let's first set a value at the key `connections` by using the `SET` command:

```
redis:6379> SET connections 10
OK
```

Let's check for the value by using the `GET` command:

```
redis:6379> GET connections
"10"
```

Now let's try if we can overwrite it by using another `SET` command:

```
redis:6379> SET connections 20
OK
redis:6379> GET connections
"20"
```

Let's see what happens if we are using the `SETNX` command. 

```
redis:6379> SETNX connections 30
(integer) 0
redis:6379> GET connections
"20"
```

if we are using `SETNX` on a key which does not yet exists, we get a different answer:

```
redis:6379> SETNX newkey 30
(integer) 1
```

Let's use `MSET` to set multiple key value pairs...

```
redis:6379> MSET key1 10 key2 20 key3 30
(integer) 1
```

and the oppposite `MGET` to get multiple values for multiple keys back. 

```
redis:6379> MGET key1 key3
1) "10"
2) "30"
```

**Note**: this is very much different to single `SET` and `GET` commands, as it it is done atomically in one operation. 

### Increment and Decrement operation

Now let's treat the value as a counter. 

First we initialise the connections value to 10, followed by a `INCR` to increment it by one. 
 
```
redis:6379> SET connections 10
OK
redis:6379> INCR connections 
(integer) 11
```

We can see that we get the new value of the counter back. 

Next we increase it by 10, using the `INCRBY` command. 

```
redis:6379> INCRBY connections 10
(integer) 21
```

Now let's do the opposite and decrement the counter value. First using the `DECR` command the counter is decremented by one. 

```
redis:6379> DECR connections
(integer) 20
```

and then with the `DECRBY` we can specify the decrement to use, here we use 10. 

```
redis:6379> DECRBY connections 10
(integer) 10
```

Now let's delete the key/value pair and see what happens if we use `INCR` on a non-existing key. 

```
redis:6379> DEL connections
(integer) 1

redis:6379> EXISTS connections
(integer) 0

redis:6379> INCR connections
(integer) 1
```

We can see that the `INCR` automatically starts with the value 0 and increments it by 1, which is the result we get back. 

----
**Note:** There is something special about INCR. Why do we provide such an operation if we can do it ourselves with a bit of code? After all it is as simple as:
x = GET count
x = x + 1
SET count x

The problem is that doing the increment in this way will only work as long as there is a single client using the key. See what happens if two clients are accessing this key at the same time:

  1.	Client A reads count as 10.
  2.	Client B reads count as 10.
  3.	Client A increments 10 and sets count to 11.
  4.	Client B increments 10 and sets count to 11.

We wanted the value to be 12, but instead it is 11! This is because incrementing the value in this way is not an atomic operation. Calling the `INCR` command in Redis will prevent this from happening, because it is an atomic operation. Redis provides many of these atomic operations on different types of data.

----

### Expiration and Time to Live

Redis can be told that a key should only exist for a certain length of time. This is accomplished with the `EXPIRE` and `TTL` commands.

First let's set a new key/value pair. 

```
redis:6379> SET resource:lock "Redis Demo"
OK
```

and then set it to expire after 2 minutes (120 seconds) by using the `EXPIRE` command.

```
redis:6379> EXPIRE resource:lock 120
(integer) 1
```

This sets the key `resource:lock` to be deleted in 120 seconds. You can test how long a key will exist for with the `TTL` command. It returns the number of seconds until it will be deleted.

```
redis:6379> TTL resource:lock
(integer) 96
```

Waiting the 96 seconds and doing the same command again we can see that it has been deleted.

```
redis:6379> TTL resource:lock
(integer) -2
```

The -2 for the TTL of the key count means that the key does (not/no longer) exist. We can prove it using the `EXISTS` command.

```
redis:6379> EXISTS resource:lock
(integer) 0
```

If you `SET` a key to a new value, its TTL will reset. Let's see that behaviour, by creating the value directly with an expiration time. This can either be done with the special `SETEX` or by `SET` and the option `EX`.  

```
redis:6379> SET resource:lock "Redis Demo 1" EX 120
OK
```

We can see that the time-to-live has been set upon creation. 

```
redis:6379> TTL resource:lock
(integer) 119
```

Now use the SET command to update the value

```
redis:6379> SET resource:lock "Redis Demo 2"
OK
```

We can see that the time-to-live has been cleared. 

```
redis:6379> TTL resource:lock
(integer) -1
```

Check the full list of [Srings commands](https://redis.io/commands#string) for more information.

##	List data structures

Redis also supports several more complex data structures. The first one we'll look at is a list. A list is a series of ordered values. Some of the important commands for interacting with lists are `RPUSH`, `LPUSH`, `LLEN`, `LRANGE`, `LPOP``, and RPOP. You can immediately begin working with a key as a list, as long as it doesn't already exist as a different type.

`RPUSH` puts the new value at the end of the list.

Let's add a new item to the end of a non-existing list called `skills` using the `RPUSH` command. 

```
redis:6379> RPUSH skills "Oracle RDBMS"
(integer) 1
```

We can see that the list now holds 1 item. Let's add another skill to the `skills` list. 

```
redis:6379> RPUSH skills "Redis"
(integer) 2
```

Not let's see the values currently in the `skills` list. Can we use the `GET` command?

```
redis:6379> GET skills
(error) WRONGTYPE Operation against a key holding the wrong kind of value
```

The `GET` command belongs to the `String` group and cannot be used for `list` structures.
But we can use the `LRANGE` command for that. 

```
redis:6379> LRANGE skills 0 -1
1) "Oracle RDBMS"
2) "Redis"
``` 

`LPUSH` puts the new value at the start of the list.

```
redis:6379> LPUSH skills "SQL Server"
(integer) 3
redis:6379> LRANGE skills 0 -1
1) "SQL Server"
2) "Oracle RDBMS"
3) "Redis"
```

`LRANGE` gives a subset of the list. It takes the index of the first element you want to retrieve as its first parameter and the index of the last element you want to retrieve as its second parameter. A value of -1 for the second parameter means to retrieve elements until the end of the list.

```
redis:6379> LRANGE skills 0 -1 
1) "SQL Server"
2) "Oracle RDBMS"
3) "Redis"
```

```
redis:6379> LRANGE skills 0 1 
1) "SQL Server"
2) "Oracle RDBMS"
```

```
redis:6379> LRANGE skills 1 2 
2) "Oracle RDBMS"
3) "Redis"
```

`LLEN` returns the current length of the list.

```
redis:6379> LLEN skills 
(integer) 3
```

`LPOP` removes the first element from the list and returns it.

```
redis:6379> LPOP skills 
"SQL Server"
```

`RPOP` removes the last element from the list and returns it.

```
redis:6379> RPOP skills 
"Redis"
```

**Note**: the list has now only one element left:

```
redis:6379> LLEN skills 
(integer) 1
redis:6379> LRANGE skills 0 -1
2) "Oracle RDBMS"
```

Check the full list of [List commands](https://redis.io/commands#list) for more information.

## Set data structures

The next data structure that we'll look at is the set. 

A set is similar to a list, except it does not have a specific order and each element may only appear once. Some of the important commands in working with sets are `SADD`, `SREM`, `SISMEMBER`, `SMEMBERS` and `SUNION`.

`SADD` adds the given value to the set. 

```
redis:6379> SADD nosql:products "Cassandra"
(integer) 1
redis:6379> SADD nosql:products "Redis"
(integer) 1
redis:6379> SADD nosql:products "MongoDB"
(integer) 1
```

`SMEMBERS` returns a list of all the members of this set.

```
redis:6379> SMEMBERS nosql:products
1) "Redis"
2) "Cassandra"
3) "MongoDB"
```

`SREM` removes the given value from the set.


```
redis:6379> SREM nosql:products "MongoDB"
(integer) 1
redis:6379> SMEMBERS nosql:products
1) "Redis"
2) "Cassandra"
```

`SISMEMBER` tests if the given value is in the set.

```
redis:6379> SISMEMBER nosql:products "Cassandra"
(integer) 1
redis:6379> SISMEMBER nosql:products "MongoDB"
(integer) 0
```

Cassandra is a member of the nosql:products, but MongoDB is not (therefore the result of 0).

`SUNION` combines two or more sets and returns the list of all elements.

first let's create another set of RDBMS products:

```
redis:6379> SADD rdbms:products "Oracle"
(integer) 1
redis:6379> SADD rdbms:products "SQL Server"
(integer) 1
```

now create the union of the two:

```
redis:6379> SUNION rdbms:products nosql:products
1) "SQL Server"
2) "Cassandra"
3) "Redis"
4) "Oracle"
```

SUNIONSTORE combines two or more sets and stores the result into a new set.

```
redis:6379> SUNIONSTORE database:products rdbms:products nosql:products
(integer) 4
redis:6379> SMEMBERS database:products
1) "SQL Server"
2) "Cassandra"
3) "Redis"
4) "Oracle"
```

`SINTER` intersects two or more sets and returns the list of the intersecting elements.

```
redis:6379> SADD favorite:products "Cassandra"
(integer) 1
redis:6379> SADD favorite:products "Oracle"
(integer) 1

redis:6379> SINTER database:products favorite:products
1) "Cassandra"
2) "Oracle"
```

Check the full list of [Set commands](https://redis.io/commands#set) for more information.
   
## Sorted Set data structures
Sets are a very handy data type, but as they are unsorted they don't work well for a number of problems. This is why Redis 1.2 introduced Sorted Sets.
A sorted set is similar to a regular set, but now each value has an associated score. This score is used to sort the elements in the set.

`ZADD` adds one or more members to a sorted set, or update its score if it already exists.

```
redis:6379> ZADD pioneers 1940 "Alan Kay"
(integer) 1
redis:6379> ZADD pioneers 1906 "Grace Hopper"
(integer) 1
redis:6379> ZADD pioneers 1953 "Richard Stallman"
(integer) 1
redis:6379> ZADD pioneers 1965 "Yukihiro Matsumoto"
(integer) 1
redis:6379> ZADD pioneers 1916 "Claude Shannon"
(integer) 1
redis:6379> ZADD pioneers 1969 "Linus Torvalds"
(integer) 1
redis:6379> ZADD pioneers 1957 "Sophie Wilson"
(integer) 1
redis:6379> ZADD pioneers 1912 "Alan Turing"
(integer) 1
```

In these examples, the scores are years of birth and the values are the names of famous people in informatics.

`ZRANGE` returns a range of members in a sorted set by index (ordered low to high, e.g. ascending), optionally also returns the scores. Index starts with 0. 

```
redis:6379> ZRANGE pioneers 2 4
1) "Claude Shannon"
2) "Alan Kay"
3) "Richard Stallman"

redis:6379> ZRANGE pioneers 2 4 WITHSCORES
1) "Claude Shannon"
2) "1916"
3) "Alan Kay"
4) "1940"
5) "Richard Stallman"
6) "1953"
```

`ZREVRANGE` returns a range of members in a sorted set by index (ordered high to low, e.g. descending), optionally also returns the scores. Index starts with 0. 

```
redis:6379> ZREVRANGE pioneers 0 2
1) "Linus Torvalds"
2) "Yukihiro Matsumoto"
3) "Sophie Wilson"
```

Check the full list of [Sorted Set commands](https://redis.io/commands#sorted_set) for more information.

## Hash structures

Simple strings, sets and sorted sets already get a lot done but there is one more data type Redis can handle: Hashes.

Hashes are maps between string fields and string values, so they are the perfect data type to represent objects (eg: A User with a number of fields like name, surname, age, and so forth):

`HSET` sets the field in the hash stored at the given key to a value. 

```
redis:6379> HSET user:1000 name "John Smith"
(integer) 1
redis:6379> HSET user:1000 email "john.smith@example.com"
(integer) 1
redis:6379> HSET user:1000 password "s3cret"
(integer) 1
```

To get back the saved data use `HGETALL` command:

```
redis:6379> HGETALL user:1000
1) "name"
2) "John Smith"
3) "email"
4) "john.smith@example.com"
5) "password"
6) "s3cret"
```

You can also set multiple fields at once using the `HMSET` command. 

```
redis:6379> HMSET user:1001 name "Mary Jones" password "hidden" email "mjones@example.com"
OK
```

let's see that this worked and a new hash as been created.

```
redis:6379> HGETALL user:1001
1) "name"
2) "Mary Jones"
3) "password"
4) "hidden"
5) "email"
6) "mjones@example.com"
```

If you only need a single field value that is possible as well using the `HGET` command

```
redis:6379> HGET user:1001 name
"Mary Jones"
```

Numerical values in hash fields are handled exactly the same as in simple strings and there are operations to increment this value in an atomic way.


```
redis:6379> HSET user:1000 visits 10
(integer) 1
redis:6379> HINCRBY user:1000 visits 1
(integer) 11
redis:6379> HINCRBY user:1000 visits 10
(integer) 21
redis:6379> HDEL user:1000 visits
(integer) 1
```
    
Check the full list of [Hash commands](https://redis.io/commands#hash) for more information.

## Redis Benchmark

```bash
docker run -it --rm --network nosql-platform bitnami/redis redis-benchmark -h redis-1 -q -n 100000
```

## Python

```
conda install redis-py
```

```
import redis
r = redis.Redis(host='redis', port=6379, db=0)
```

```
r.ping()
```

```
r.set('foo','bar')
```


