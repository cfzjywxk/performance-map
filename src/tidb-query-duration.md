# TiDB query duration

```railroad
Diagram(
  NonTerminal("Receive packet from the client", {href: "#receive-packet-from-the-client"}),
  NonTerminal("Process query in TiDB (include TiKV)", {href: "#process-query-in-tidb-include-tikv"}),
)
```

- **Receive packet from the client**: The duration contains network and operation schedule latency, etc.
- **Process query in TiDB (include TiKV)**: Include all the duration TiDB executing queries and the duration sending response back to the client.

# Receive packet from the client

```railroad
Diagram(
  NonTerminal("Write system call duration in client"),
  NonTerminal("Network duration"),
  NonTerminal("Parsing TCP protocol duration"),
  NonTerminal("Schedule TiDB conn duration"),
  NonTerminal("Copy packet into user space duration"),
)
```

- **Write system call duration in client**: Client send request into NIC by write system call.
- **Network duration**: Network transmission duration.
- **Parsing TCP protocol duration**: After server's NIC receiving the TCP request, system need to parse the packet for application usage.
- **Schedule TiDB conn duration**: There are some cases that TiDB does not read packet from the connection immediately, such duration starts from writing OK package for last query and ends when it's ready to read the packet for current query.
- **Copy packet into user space duration**: TiDB read packet from connection by read system call which will copy the packet into user space.

# Process query in TiDB (include TiKV)

```railroad
Diagram(
  NonTerminal("Token wait duration", {cls: "with-metrics"}),
  Choice(
    0,
    Comment("Prepared statement"),
    NonTerminal("Parse duration", {cls: "with-metrics"}),
  ),
  OneOrMore(
    Sequence(
      Choice(
        0,
        NonTerminal('Optimize prepared plan duration'),
        Sequence(
          Comment('Plan cache miss'),
          NonTerminal('Compile duration', {cls: "with-metrics"}),
        ),
      ),
      NonTerminal("TSO wait duration", {cls: "with-metrics"}),
      NonTerminal("Execution duration", {href: "#execution-duration", cls: "with-metrics"}),
    ),
    Comment("Retry"),
  ),
)
```

- **Token wait duration**: Flow control for all commands in TiDB, it's extremely fast when there are enough tokens in the pool.
- **Parse duration**: Parse sql query text to AST (when using text protocol). For simplicity, we assume 1 command contains 1 sql only.
- **Compile duration**: Optimize query and compile it to physical plan is required.
- **Optimize prepared plan duration**: The cached plan need to be re-checked in plan cache. This optimize should be fast in most cases, and it'll be re-compiled when it's not valid, e.g. schema has been changed.
- **TSO wait duration**: TSO will be resolved in executor builder phase, the in-txn current read(insert/delete/update/selectForUpdate) statements will wait for the latest TSO here.
- **Execution duration**: This phase contains writing result to the client, and there are many different possible paths, it's significantly different for read/write queries.
- **Retry**: Some queries may suffer from failure and they are retryable(due to both logical and system failures), if the retry is successful, the failure duration will be part of latency.

# Execution duration

```railroad
Diagram(
  Choice(
    0,
    Sequence(
      Comment("Read"),
      OneOrMore(
        Sequence(
          NonTerminal("Read next"),
          NonTerminal("Write result to client"),
        ),
        Comment("drain the result set"),
      ),
    ),
    Sequence(
      Comment("Write"),
      NonTerminal("Execute write"),
      Choice(
        0,
        NonTerminal("Pessimistic lock"),
        Comment("Optimisitc mode"),
      ),
      Choice(
        0,
        NonTerminal("Commit"),
        Comment("Explicit transaction"),
      ),
      NonTerminal("Write result to client"),
    ),
  ),
)
```

- Write executors are executed immediately, and a result with affected rows need to be written to client, such write will execute only once for one query.
- Pessimistic write will have an extra phase that acquires pessimistic locks.
- Auto-commit write queries will be committed after executing, in which 2PC will be executed.
- The resultset of read executor will be fetched and written to client in batches, this strategy makes the diagnosis of read latency hard. Some slow async requests may not affect the user-end latency if they are ready before the Next of it is called. You need to carefully analyze such issues by digging into some concrete query and plan.
