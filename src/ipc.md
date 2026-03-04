# Q IPC Deep Dive: The Protocol, the Patterns, and the Pitfalls

The IPC protocol is the foundation everything in this book rests on. Every library we've discussed — qPython, pykx, kdbplus for Rust, rkdb for R — is ultimately speaking this protocol. Understanding it once means you can debug integration issues at the wire level, write your own client in any language, and understand why certain patterns are faster or safer than others.

## The Wire Protocol

Q's IPC uses a simple binary protocol over TCP. A message has a fixed 8-byte header followed by a serialized Q value.

### Header Structure

```
Byte 0: Endianness (0x01 = little-endian, 0x00 = big-endian)
Byte 1: Message type
         0x00 = async (fire and forget)
         0x01 = sync (request, expects response)
         0x02 = response (reply to sync request)
Byte 2: Compression flag (0x01 if body is compressed)
Byte 3: Reserved (0x00)
Bytes 4-7: Total message length (little-endian int32, includes header)
```

After the header: serialized Q data in KDB's internal binary format.

You can observe this with a raw TCP dump:

```bash
# Start q on port 5001, connect with nc, and observe the handshake
# (The connection handshake sends/receives capability strings)
```

Or from Q itself, capture the raw bytes:

```q
/ Serialize a Q value to bytes
-8! 42              / integer 42 as byte sequence
/ Result: 0x0100000009000000fa2a000000

/ Deserialize
-9! 0x0100000009000000fa2a000000
/ Result: 42
```

The `-8!` (serialize) and `-9!` (deserialize) system functions are your wire-format debugging tools. Every Q value has a deterministic binary representation.

### Serialization Format

```q
/ See the raw bytes of various types
-8! 3.14            / float: 0x010000000d000000070182d497855e0940
-8! `hello          / symbol: 0x01000000100000000568656c6c6f00
-8! "hello"         / char list: 0x010000000e00000006000500000068656c6c6f
-8! 1 2 3           / long list: 0x010000001800000006000300000001000000000000000200000000000000...
-8! ([] a:1 2; b:`x`y)  / table
```

Type bytes: `0x01` = boolean, `0x04` = byte, `0x05` = short, `0x06` = int, `0x07` = long, `0x08` = real, `0x09` = float, `0x0a` = char, `0x0b` = symbol. Negative values (`0xfb`, `0xfc`...) are atom equivalents of the corresponding list types.

You don't need to implement this by hand (the libraries do it), but knowing it exists means you can debug with Wireshark or `tcpdump` when something goes wrong.

## Handles and Connection Lifecycle

```q
/ Opening a handle
h: hopen `::5001          / localhost:5001, no auth
h: hopen `::5001:user:pass / with auth
h: hopen (`:myserver; 5001; "user:pass"; 10000)  / with timeout (ms)

/ Check handle type
type h    / -7h = int handle

/ Send synchronous message (blocking)
result: h "select count from trade where date=.z.d"

/ Send asynchronous message (non-blocking, no response)
neg[h] "slow_background_task[]"

/ Close
hclose h

/ Check if handle is open
.z.W    / dictionary of open handles -> their send buffer size
h in key .z.W  / true if h is open
```

### Timeouts

Q's default IPC has no timeout — a hung remote process will block your q process indefinitely. For production code, always use timeout:

```q
/ Synchronous call with timeout (kdb+ 4.1+)
h: hopen (`:server; 5001; ""; 5000)   / 5 second connect timeout

/ Per-query timeout using .Q.trp
.Q.trp[{h "slow_query[]"}; (); {[err] 'err}]

/ Or with system-level timeout control
system "T 30"   / set 30 second IPC timeout globally (not recommended)
```

The timeout on `hopen` is a connection timeout, not a query timeout. For query timeouts, you need either a gateway that enforces them or the async pattern described below.

## Synchronous vs Asynchronous: The Key Trade-off

### Synchronous (Default)

```q
/ Client sends query, blocks until response
result: h "count trade"
/ h is blocked during query execution — can't send another query on this handle
```

Simple, safe, familiar. The bottleneck: one outstanding request per handle. For high-throughput scenarios, you need either multiple handles or the async pattern.

### Asynchronous (Fire-and-Forget)

```q
/ Client sends message, returns immediately, no response
neg[h] ".u.upd[`trade; data]"
```

Used by tickerplants to publish updates to subscribers. Very fast — no round-trip. But: if the message errors on the server, you'll never know. Use for pub/sub and logging, not for queries that need results.

### Async with Callback (The Advanced Pattern)

kdb+ 4.1 introduced deferred sync, which gives you async queries that still get responses:

```q
/ Server-side setup: handle the deferred callback
.z.ps: {[x]
    / x is the incoming async message
    / For deferred sync, x is (neg handle; function; args)
    if[10h = type x;
        / This is a string eval, just execute
        @[value; x; {[e] 'e}];
        :()
    ];
    / Deferred pattern: (response_handle; function; args)
    if[99h = type x;
        h: neg x 0;           / response handle
        fn: x 1;              / function
        args: x 2;            / arguments
        result: @[fn; args; {[e] 'e}];
        h result;             / send result back asynchronously
        :()
    ]
    }

/ Client-side: send async, receive via .z.ps on client
/ The client becomes a listener too
.z.ps: {[x] show x}   / client's async handler

/ Send deferred request:
neg[h] (neg[.z.w]; `myFunction; (`AAPL; 2024.01.15))
/ The server calls myFunction, sends result back to client async
```

This is how you build non-blocking, high-throughput query patterns. The complication is that your client now needs a message loop.

## Gateway Patterns

A gateway is a q process that sits in front of other q processes — routing queries, merging results, managing permissions. The canonical gateway patterns:

### Simple Synchronous Gateway

```q
/ gateway.q
\p 5000

/ Backend connections
backends: `hdb`rdb!5001 5002;
handles: hopen each `::/:' string backends;

/ Route queries to appropriate backend
route: {[query]
    / Simple routing: queries mentioning historical tables → hdb
    $["hdb" in query;
        handles`hdb;
        handles`rdb] "\"", query, "\""
    }

/ Query handler
.z.pg: {[x]
    h: route x;
    h x  / forward and return result
    }
```

### Scatter-Gather: Fan-Out Across Partitions

The pattern for querying across multiple kdb+ processes and merging results:

```q
/ Query multiple processes, merge results
scatterQuery: {[handles; query]
    / Send async to all, collect responses
    results: handles @\: query;
    / Merge (appropriate for tables)
    raze results
    }

/ Example with multiple date-partitioned HDB processes
hdb_handles: `::5001`::5002`::5003;  / each has different date ranges
h1: hopen `::5001;
h2: hopen `::5002;
h3: hopen `::5003;

merged: raze {h "select from trade where date=.z.d"} each (h1; h2; h3)
```

For true parallel execution, use the async pattern:

```q
/ Fan-out asynchronously
fanOut: {[handles; query]
    / Send to all without waiting
    neg[handles] @\: query;

    / Collect responses
    results: ();
    do[count handles;
        results,: enlist .z.ps[]  / wait for each response
    ];

    raze results
    }
```

### Protected Evaluation

Always wrap remote execution in protected eval to prevent server crashes from propagating:

```q
/ Server-side: protect query execution
.z.pg: {[x]
    / Protected execution
    result: @[value; x; {[err] (`error; err)}];

    / Check if error
    $[(`error ~ type result) and 2 = count result;
        '"remote error: ", string result 1;
        result]
    }
```

The `@[f; args; err_handler]` pattern is Q's try-catch. Use it on any query that comes from an external source.

## Message Handlers: The Full Set

These are the callbacks Q calls when IPC events occur:

```q
.z.pg    / synchronous GET handler (client sent sync message)
.z.ps    / asynchronous message handler (client sent async message)
.z.ph    / HTTP GET handler
.z.pp    / HTTP POST handler
.z.pw    / authentication handler — return 1b to allow, 0b to deny
.z.po    / open handler — called when a handle is opened
.z.pc    / close handler — called when a handle is closed
.z.wo    / websocket open handler
.z.wc    / websocket close handler
.z.ws    / websocket message handler
```

### Connection Tracking

```q
/ Track active connections
conns: ([handle: `long$()] host: `(); user: `(); opened: `timestamp$())

.z.po: {[h]
    / h is the new handle
    `conns upsert (h; .z.a; .z.u; .z.p)
    }

.z.pc: {[h]
    / h is the closing handle
    delete from `conns where handle=h
    }

/ See who's connected
conns
/ handle | host        user  opened
/ -------+-----------------------------------------
/ 8      | 127.0.0.1   alice 2024.01.15T09:30:01
/ 9      | 10.0.0.1    bob   2024.01.15T09:30:15
```

### Authentication

```q
/ .z.pw: called on connection attempt
/ Arguments: username (symbol), password (string/symbol)
/ Return: 1b to allow, 0b to deny

/ Simple user/password table
users: ([ user:`alice`bob`readonly]
         pass:("secret1"; "secret2"; "readpass");
         perms:`admin`admin`read)

.z.pw: {[user; pass]
    $[user in key users;
        pass ~ users[user; `pass];
        0b]
    }

/ Role-based: store role in connection handle metadata
.z.pg: {[x]
    / Check permission for this user's role
    user: .z.u;
    role: $[user in key users; users[user; `perms]; `none];

    / Block write operations for read-only users
    if[role ~ `read;
        if[any x like/: ("insert*"; "delete*"; "update*"; "*upsert*");
            '"permission denied"]
    ];

    value x
    }
```

## Compression

For large result sets, Q can compress IPC messages:

```q
/ Enable compression for this connection (server-side)
/ Set compression level for outgoing messages
h set `compression`level!(1b; 6)  / zlib compression, level 6

/ Or configure globally
.z.pg: {[x]
    result: value x;
    / Compress results larger than 100KB
    $[-1000000 < type result;
        result;  / small result, no compression
        -19! result]  / compress
    }
```

Compression makes sense for large tables sent over WAN. For intranet queries, the CPU cost often outweighs the bandwidth saving.

## Common Pitfalls

### The Blocking Handle Problem

```q
/ Wrong: using one handle for high-frequency queries
/ This serializes all requests
h: hopen `::5001
do[1000; result: h "expensive_query[]"]

/ Right: connection pool or async pattern
/ Multiple handles, round-robin
handles: hopen each 1000 # `::5001
idx: 0
getHandle: {handles idx mod: count handles; idx+:1; handles idx-1}
```

### Message Size Limits

Q has a default message size limit of 2GB. For very large result sets, either:
- Paginate in Q: `select[-1000]` gets last 1000 rows; `select[1000;offset]` for pagination
- Stream results in chunks
- Compress the message

### Handle Leaks

Forgetting to close handles is a slow resource leak:

```q
/ Dangerous: handle opened but may not be closed on error
result: (hopen `::5001) "query"
/ If query errors, handle stays open

/ Safe pattern
safeQuery: {[host; port; query]
    h: hopen (host; port; ""; 5000);
    result: @[h; query; {[h; e] hclose h; 'e}[h]];
    hclose h;
    result
    }
```

### Endianness

Q handles endianness negotiation automatically during the connection handshake. You don't need to worry about it unless you're implementing the protocol from scratch — at which point you need to read the KX documentation on the wire format carefully. Most people are not doing this.

## Debugging IPC Issues

```q
/ See all open handles
.z.W    / dict: handle -> buffer size

/ Handle information
h       / just the integer
-1000 * h  / this is a trick: negative handle for async
type h  / should be -7h

/ Test connectivity without a real query
h "1b"  / returns 1b on success, errors on failure

/ Check what's waiting in the send buffer
.z.W h  / bytes waiting to be sent on handle h

/ Connection details about the *current* caller
.z.a    / caller's IP address (within .z.pg / .z.ph)
.z.u    / caller's username
.z.w    / caller's handle (use neg[.z.w] to respond async)
```

For deep debugging, start q with verbose logging:

```bash
q -p 5001 -t 1000 2>&1 | tee q.log
```

The `-t` flag enables timer callbacks; the combined output to a log file gives you a timestamped record of messages and errors.
