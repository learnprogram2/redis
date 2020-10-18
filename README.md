This README is just a fast *quick start* document. You can find more detailed documentation at [redis.io](https://redis.io).

What is Redis?
--------------

Redis is often referred as a *data structures* server. What this means is that Redis provides access to mutable data structures via a set of commands, which are sent using a *server-client* model with TCP sockets and a simple protocol. So different processes can query and modify the same data structures in a shared way.

Data structures implemented into Redis have a few special properties:

* Redis cares to store them on disk, even if they are always served and modified into the server memory. This means that Redis is fast, but that is also non-volatile.
* Implementation of data structures stress on memory efficiency, so data structures inside Redis will likely use less memory compared to the same data structure modeled using an high level programming language.
* Redis offers a number of features that are natural to find in a database, like replication, tunable levels of durability, cluster, high availability.

Another good example is to think of Redis as a more complex version of memcached, where the operations are not just SETs and GETs, but operations to work with complex data types like Lists, Sets, ordered data structures, and so forth.

If you want to know more, this is a list of selected starting points:

* Introduction to Redis data types. http://redis.io/topics/data-types-intro
* Try Redis directly inside your browser. http://try.redis.io
* The full list of Redis commands. http://redis.io/commands
* There is much more inside the Redis official documentation. http://redis.io/documentation

Building Redis
--------------

巴拉巴拉的, 用 C build, 我们用不到

Allocator
---------

redis默认用的`MALLOC`做的内存分配, Linux上面用的`jemalloc`. 设置`MALLOC` environment variable, 可以更改

To force compiling against libc malloc, use:

    % make MALLOC=libc

Running Redis
-------------

To run Redis with the default configuration just type:

    % cd src
    % ./redis-server

If you want to provide your redis.conf, you have to run it using an additional
parameter (the path of the configuration file):

    % cd src
    % ./redis-server /path/to/redis.conf

It is possible to alter the Redis configuration by passing parameters directly
as options using the command line. Examples:

    % ./redis-server --port 9999 --replicaof 127.0.0.1 6379
    % ./redis-server /etc/redis/6379.conf --loglevel debug

All the options in redis.conf are also supported as options using the command
line, with exactly the same name.


Redis internals: Redis 源码介绍
===

If you are reading this README you are likely in front of a Github page
or you just untarred the Redis distribution tar ball. In both the cases
you are basically one step away from the source code, so here we explain
the Redis source code layout, what is in each file as a general idea, the
most important functions and structures inside the Redis server and so forth.
We keep all the discussion at a high level without digging into the details
since this document would be huge otherwise and our code base changes
continuously, but a general idea should be a good starting point to
understand more. Moreover most of the code is heavily commented and easy
to follow.

Source code layout
---

The Redis root directory just contains this README, the Makefile which
calls the real Makefile inside the `src` directory and an example
configuration for Redis and Sentinel. You can find a few shell
scripts that are used in order to execute the Redis, Redis Cluster and
Redis Sentinel unit tests, which are implemented inside the `tests`
directory.

Inside the root are the following important directories:

* `src`: contains the Redis implementation, written in C.
* `tests`: contains the unit tests, implemented in Tcl.
* `deps`: contains libraries Redis uses. Everything needed to compile Redis is inside this directory; your system just needs to provide `libc`, a POSIX compatible interface and a C compiler. Notably `deps` contains a copy of `jemalloc`, which is the default allocator of Redis under Linux. Note that under `deps` there are also things which started with the Redis project, but for which the main repository is not `antirez/redis`. An exception to this rule is `deps/geohash-int` which is the low level geocoding library used by Redis: it originated from a different project, but at this point it diverged so much that it is developed as a separated entity directly inside the Redis repository.

There are a few more directories but they are not very important for our goals
here. We'll focus mostly on `src`, where the Redis implementation is contained,
exploring what there is inside each file. The order in which files are
exposed is the logical one to follow in order to disclose different layers
of complexity incrementally.

Note: lately Redis was refactored quite a bit. Function names and file
names have been changed, so you may find that this documentation reflects the
`unstable` branch more closely. For instance in Redis 3.0 the `server.c`
and `server.h` files were named `redis.c` and `redis.h`. However the overall
structure is the same. Keep in mind that all the new developments and pull
requests should be performed against the `unstable` branch.

server.h  存储着 data structures.
---

All the server configuration and in general all the shared state is
defined in a global structure called `server`, of type `struct redisServer`.
A few important fields in this structure are:

* `server.db` is an array of Redis databases, where data is stored.
* `server.commands` is the command table.
* `server.clients` is a linked list of clients connected to the server.
* `server.master` is a special client, the master, if the instance is a replica.

There are tons of other fields. Most fields are commented directly inside
the structure definition.

Another important Redis data structure is the one defining a client.
In the past it was called `redisClient`, now just `client`. The structure
has many fields, here we'll just show the main ones:

    struct client {
        int fd;
        sds querybuf;
        int argc;
        robj **argv;
        redisDb *db;
        int flags;
        list *reply;
        char buf[PROTO_REPLY_CHUNK_BYTES];
        ... many other fields ...
    }

The client structure defines a *connected client*:

* The `fd` field is the client socket file descriptor.
* `argc` and `argv` are populated with the command the client is executing, so that functions implementing a given Redis command can read the arguments.
* `querybuf` accumulates the requests from the client, which are parsed by the Redis server according to the Redis protocol and executed by calling the implementations of the commands the client is executing.
* `reply` and `buf` are dynamic and static buffers that accumulate the replies the server sends to the client. These buffers are incrementally written to the socket as soon as the file descriptor is writable.

As you can see in the client structure above, arguments in a command
are described as `robj` structures. The following is the full `robj`
structure, which defines a *Redis object*:

    typedef struct redisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;

redisObject使用`type`和`encoding`表示所有Redis的基本数据类型和具体的存储结构.
比如: strings, lists, sets, sorted sets and so forth.
a `refcount`, so that the same object can be referenced in multiple places 
without allocating it multiple times. Finally the `ptr` field points to the 
actual representation of the object.

为了避免redisObject(robj)的重定向太多消耗, 现在在很多地方可以直接 use plain dynamic 
strings not wrapped inside a Redis object.

server.c: Redis server 的 main 方法
---

redis server的主要启动步骤: 

* `initServerConfig()` setups the default values of the `server` structure.
                初始化`server`数据结构的默认值.
* `initServer()` allocates the data structures needed to operate, setup the listening socket, and so forth.
                分配其它的工具数据结构, 比如socket....
* `aeMain()` starts the event loop which listens for new connections.
                开始监听事件了.

There are two special functions called periodically by the event loop: 在ae.c里面
在ae.c的事件监听里, 主要干两个事情:

1. `serverCron()` is called periodically (according to `server.hz` frequency), and performs tasks that must be performed from time to time, like checking for timedout clients.
    定时任务
2. `beforeSleep()` is called every time the event loop fired, Redis served a few requests, and is returning back into the event loop.

server.c里面还有其它的方法: 

* `call()` is used in order to call a given command in the context of a given client.
* `activeExpireCycle()` handles eviciton of keys with a time to live set via the `EXPIRE` command.
* `freeMemoryIfNeeded()` is called when a new write command should be performed but Redis is out of memory according to the `maxmemory` directive.
* The global variable `redisCommandTable` defines all the Redis commands, specifying the name of the command, the function implementing the command, the number of arguments required, and other properties of each command.

networking.c: 存储着所有的IO操作: master, client, replicas
---

* `createClient()` allocates and initializes a new client. 初始化一个新的Client.
* `addReply*()` 很多方法, 往client的output buffer里面添加各种类型的数据
* `writeToClient()` 把outputbuffer的数据发给client,  is called by the *writable event handler* `sendReplyToClient()`.
* `readQueryFromClient()` is the *readable event handler* and accumulates data from read from the client into the query buffer.
* `processInputBuffer()` 根据redis协议解析query_buffer的数据, 解析出来一个command, 就调用`server.processCommand()` 执行.
* `freeClient()` deallocates, disconnects and removes a client.

aof.c and rdb.c
---

As you can guess from the names these files implement the RDB and AOF
persistence for Redis. Redis uses a persistence model based on the `fork()`
system call in order to create a thread with the same (shared) memory
content of the main Redis thread. This secondary thread dumps the content
of the memory on disk. This is used by `rdb.c` to create the snapshots
on disk and by `aof.c` in order to perform the AOF rewrite when the
append only file gets too big.

The implementation inside `aof.c` has additional functions in order to
implement an API that allows commands to append new commands into the AOF
file as clients execute them.
`aof.c`文件里是实现了一些把命令追加到AOF文件的方法, 在调用 `call()` 执行命令的时候就追加.

db.c:  DB的API
---

Certain Redis commands operate on specific data types, others are general.
Examples of generic commands are `DEL` and `EXPIRE`. They operate on keys
and not on their values specifically. All those generic commands are
defined inside `db.c`.
general 操作的API. 是对key的操作, 不会具体的改变存储的value

Moreover `db.c` implements an API in order to perform certain operations
on the Redis dataset without directly accessing the internal data structures.
`db.c`里面的API包装了redis的具体数据结构操作.

主要的方法就是: 其它的就是具体的实现:

* `lookupKeyRead()` and `lookupKeyWrite()`: 根据key查找值
* `dbAdd()` 创建key
* `dbDelete()` removes a key and its associated value.
* `emptyDb()` removes an entire single database or all the databases defined.

object.c: operations on `robj`
---

The `robj` structure defining Redis objects was already described. Inside
`object.c` there are all the functions that operate with Redis objects at
a basic level, like functions to allocate new objects, handle the reference
counting and so forth. 
`object.c`里面包含着redis对象 `robj` 的基本operate方法.

需要注意的两个方法:

* `incrRefcount()` and `decrRefCount()` 引用计数维护, 0就直接 free掉
* `createObject()` allocates a new object. There are also specialized functions to allocate string objects having a specific content, like `createStringObjectFromLongLong()` and similar functions.

This file also implements the `OBJECT` command.
对object的操作暴露出去, 实现了`object`命令.

replication.c
---

This is one of the most complex files inside Redis, it is recommended to
approach it only after getting a bit familiar with the rest of the code base.
In this file there is the implementation of both the master and replica role
of Redis.
很复杂, 先搞明白其它的. 实现了redis里面master和replica的两个角色的操作.

One of the most important functions inside this file is `replicationFeedSlaves()` 
that writes commands to the clients representing replica instances connected
to our master, so that the replicas can get the writes performed by the clients:
this way their data set will remain synchronized with the one in the master.
`replicationFreedSlaves()` 用于向slave之类的replica角色写命令. replicas拿到数据.

This file also implements both the `SYNC` and `PSYNC` commands that are
used in order to perform the first synchronization between masters and
replicas, or to continue the replication after a disconnection.
还实现了同步异步两种同步机制.

Other C files
---

* `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c` and `t_zset.c` contains the implementation of the Redis data types. They implement both an API to access a given data type, and the client commands implementations for these data types.
    五种基本数据结构: 是处理数据的API实现

* `ae.c` implements the Redis event loop, it's a self contained library which is simple to read and understand.
* `sds.c` is the Redis string library, check http://github.com/antirez/sds for more information.
* `anet.c` is a library to use POSIX networking in a simpler way compared to the raw interface exposed by the kernel.
* `dict.c` is an implementation of a non-blocking hash table which rehashes incrementally.
* `scripting.c` implements Lua scripting. It is completely self contained from the rest of the Redis implementation and is simple enough to understand if you are familar with the Lua API.

* `cluster.c` implements the Redis Cluster. Probably a good read only after being very familiar with the rest of the Redis code base. If you want to read `cluster.c` make sure to read the [Redis Cluster specification][3].
    redisCluter 要在熟悉了redis的运行机制后再看, 可以先看redis cluster的说明


[3]: http://redis.io/topics/cluster-spec



Anatomy of a Redis command
---

第一步: redisCommandTable[]里面放着命令string和server.h里面的命令API.

第二步: 在networkling.c里面processInputBuffer的时候, 收到了一条命令就会processCommand, 
    会去redisServer这个对象里面找command, 这个时候就用上了.

第三步: 处理完了之后, 就会把返回值用addReply()系列方法放回到output buffer里面

All the Redis commands are defined in the following way:

    void foobarCommand(client *c) {
        printf("%s",c->argv[1]->ptr); /* Do something with the argument. */
        addReply(c,shared.ok); /* Reply something to the client. */
    }

The command is then referenced inside `server.c` in the command table:

    {"foobar",foobarCommand,2,"rtF",0,NULL,0,0,0,0,0},

In the above example `2` is the number of arguments the command takes,
while `"rtF"` are the command flags, as documented in the command table
top comment inside `server.c`.

After the command operates in some way, it returns a reply to the client,
usually using `addReply()` or a similar function defined inside `networking.c`.

There are tons of commands implementations inside the Redis source code
that can serve as examples of actual commands implementations. To write
a few toy commands can be a good exercise to familiarize with the code base.

There are also many other files not described here, but it is useless to
cover everything. We want to just help you with the first steps.
Eventually you'll find your way inside the Redis code base :-)

谢谢! 非常感谢写这个readMe的人, thanks a lots.

Enjoy!
