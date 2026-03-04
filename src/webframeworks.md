# Web Frameworks and Q: Serving Data to the Outside World

Q has a built-in HTTP server. This is both better and worse than you expect.

It's better because it works, it's fast, and it's already running alongside your q process with a one-line configuration change. It's worse because it's not a web framework — it's a raw HTTP server with a callback, and if you want routing, middleware, authentication, JSON serialization, and all the other things a real web framework provides, you're either building them or wrapping q with something external.

The two viable approaches:

1. **q's built-in HTTP server**: Minimal, fast, good for simple APIs and dashboards. Surprisingly capable when you understand how it works.
2. **External web framework wrapping q via IPC**: Python (FastAPI), Node.js, Go — your choice — sits in front of kdb+ and translates HTTP requests to IPC calls. More operational overhead, much more flexibility.

## Q's Built-In HTTP Server

Start q with a port and it also accepts HTTP:

```bash
q -p 5001
```

That's it. The process now accepts both Q IPC connections and HTTP requests on the same port. An HTTP GET to `localhost:5001?query=2+2` returns... something. By default it returns Q's text representation, which is approximately useful.

### The `.z.ph` Handler

`.z.ph` is the HTTP GET handler. It receives the raw query string and returns whatever you give it:

```q
/ Default .z.ph strips the query and evaluates it
/ (this is the built-in behavior, not something you write)

/ Override with your own handler
.z.ph: {[x]
    / x is the raw HTTP request as a string
    / Parse the query parameter
    qry: .h.mp x;          / parse into a dict of params

    / Execute if a 'q' parameter is present
    result: $["q" in key qry;
        .Q.s value qry`q;  / evaluate and format
        "No query parameter"];

    / Return HTTP response
    "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\n", result
    }
```

### Serving JSON

The moment where this becomes actually useful — returning JSON for real API consumers:

```q
/ JSON serialization/deserialization (built-in since kdb+ 3.x)
.j.j   / serialize to JSON
.j.k   / deserialize from JSON

/ Simple JSON API endpoint
.z.ph: {[x]
    / Parse query parameters
    params: .h.mp x;

    / Route based on 'endpoint' parameter
    response: $[
        "endpoint" in key params;
        handleRequest params;
        "404 Not Found"
    ];

    "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\nAccess-Control-Allow-Origin: *\r\n\r\n",
    .j.j response
    }

handleRequest: {[params]
    ep: params`endpoint;
    $[ep ~ "trades";   getTrades params;
      ep ~ "quotes";   getQuotes params;
      ep ~ "summary";  getSummary params;
      (`error; "Unknown endpoint")]
    }

getTrades: {[params]
    sym: `$params`sym;
    dt: "D"$params`date;
    select time, sym, price, size
    from trade
    where date=dt, sym=sym
    }
```

Test with curl:
```bash
curl "http://localhost:5001?endpoint=trades&sym=AAPL&date=2024.01.15"
```

### POST Handler and `.z.pp`

For POST requests with JSON bodies:

```q
.z.pp: {[x]
    / x contains the HTTP headers and body
    / Parse the JSON body
    body: .j.k last "\r\n\r\n" vs x;

    / Handle the request
    result: processPost body;

    "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n",
    .j.j result
    }

processPost: {[body]
    / body is now a Q dictionary
    $[`action in key body;
        executeAction body;
        `error`message!("missing_field"; "action is required")]
    }

executeAction: {[body]
    action: body`action;
    $[action ~ "insert";   insertRecord body;
      action ~ "delete";   deleteRecord body;
      `error`message!("unknown_action"; action)]
    }
```

### A Real Example: OHLC API

```q
/ OHLC data API — the kind of thing a dashboard or charting library calls

.z.ph: {[x]
    params: .h.mp x;

    result: @[handleApi; params; {[e] `status`error!("error"; string e)}];

    headers: "HTTP/1.1 200 OK\r\n";
    headers,: "Content-Type: application/json\r\n";
    headers,: "Access-Control-Allow-Origin: *\r\n\r\n";

    headers, .j.j result
    }

handleApi: {[params]
    / Validate required parameters
    required: `sym`date;
    missing: required where not required in key params;
    if[count missing; '"missing parameters: ", " " sv string missing];

    sym:  `$params`sym;
    dt:   "D"$params`date;
    bins: `long$@[value; `$"$[`bins in key params; params`bins; "78"]; 78];

    / OHLC in time buckets
    ohlc: select
        open:  first price,
        high:  max price,
        low:   min price,
        close: last price,
        volume: sum size,
        vwap:  size wavg price
    from trade
    where date=dt, sym=sym
    by time: bins xbar time.minute;

    / Convert to list-of-records for JSON
    `sym`date`bars!(sym; dt; flip ohlc)
    }
```

Calling this from JavaScript:
```javascript
const response = await fetch(
  'http://localhost:5001?sym=AAPL&date=2024.01.15&bins=30'
);
const data = await response.json();
// data.bars contains {time: [...], open: [...], high: [...], ...}
```

### Embedded HTML Dashboard

q can serve HTML directly. The `.h` namespace has utilities for generating HTML tables:

```q
.z.ph: {[x]
    params: .h.mp x;

    $["page" in key params;
        servePage params`page;
        serveIndex[]
    ]}

serveIndex: {
    html: "<html><body>";
    html,: "<h1>kdb+ Dashboard</h1>";
    html,: "<p><a href='?page=trades'>Trades</a> | ";
    html,: "<a href='?page=summary'>Summary</a></p>";
    html,: "</body></html>";
    "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n", html
    }

servePage: {[page]
    data: $[page ~ "trades";
        select[-100] from trade where date=.z.d;
        page ~ "summary";
        select last price, sum size by sym from trade where date=.z.d;
        ([]) "Unknown page"];

    html: "<html><body>", .h.hb[`p; page], .h.htc[`table; .h.ht data], "</body></html>";
    "HTTP/1.1 200 OK\r\nContent-Type: text/html\r\n\r\n", html
    }
```

`.h.ht` renders a Q table as an HTML table. It's not beautiful but it works and it's one function call.

## FastAPI as a Gateway

For production APIs with real requirements (authentication, rate limiting, request validation, proper error handling), put a Python FastAPI server in front of kdb+:

```
Client → FastAPI → kdb+ (IPC)
```

The FastAPI layer handles HTTP concerns; kdb+ handles data concerns.

```python
# gateway.py
from fastapi import FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from typing import Optional
import pykx as kx
from datetime import date
import os

# Connection pool (simplified)
q_connections = []
Q_HOST = os.getenv("KDB_HOST", "localhost")
Q_PORT = int(os.getenv("KDB_PORT", "5001"))

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: create connections
    for _ in range(4):
        conn = kx.SyncQConnection(host=Q_HOST, port=Q_PORT)
        q_connections.append(conn)
    yield
    # Shutdown: close connections
    for conn in q_connections:
        conn.close()

app = FastAPI(
    title="kdb+ Gateway API",
    description="REST API for kdb+ time-series data",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET"],
    allow_headers=["*"],
)

def get_q():
    """Round-robin connection from pool."""
    import random
    return random.choice(q_connections)

@app.get("/trades")
async def get_trades(
    sym: str = Query(..., description="Symbol, e.g. AAPL"),
    date: date = Query(..., description="Date in YYYY-MM-DD format"),
    limit: int = Query(1000, ge=1, le=10000)
):
    """Get trades for a symbol on a date."""
    q = get_q()
    try:
        result = q(
            "{[s;d;n] select[-n] time, sym, price, size from trade where date=d, sym=s}",
            kx.SymbolAtom(sym),
            kx.DateAtom(date),
            kx.LongAtom(limit)
        )
        return result.pd().to_dict(orient="records")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/ohlc")
async def get_ohlc(
    sym: str = Query(..., description="Symbol"),
    date: date = Query(..., description="Date"),
    bins: int = Query(30, ge=1, le=390, description="Minutes per bar")
):
    """Get OHLC bars for a symbol."""
    q = get_q()
    try:
        result = q(
            """{[s;d;b]
                select
                    open:  first price,
                    high:  max price,
                    low:   min price,
                    close: last price,
                    volume: sum size,
                    vwap:  size wavg price
                from trade
                where date=d, sym=s
                by time: b xbar time.minute
            }""",
            kx.SymbolAtom(sym),
            kx.DateAtom(date),
            kx.LongAtom(bins)
        )
        df = result.pd()
        df.index = df.index.astype(str)  # serialize time index
        return df.reset_index().to_dict(orient="records")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/symbols")
async def get_symbols():
    """List available symbols."""
    q = get_q()
    try:
        syms = q("exec distinct sym from trade where date=.z.d")
        return {"symbols": [str(s) for s in syms.py()]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    """Health check — also tests kdb+ connectivity."""
    q = get_q()
    try:
        result = q("1b")
        return {"status": "ok", "kdb_connected": bool(result.py())}
    except Exception as e:
        return {"status": "degraded", "error": str(e)}
```

Run it:
```bash
pip install fastapi uvicorn pykx
uvicorn gateway:app --host 0.0.0.0 --port 8000 --workers 4
```

Test:
```bash
curl "http://localhost:8000/trades?sym=AAPL&date=2024-01-15"
curl "http://localhost:8000/ohlc?sym=AAPL&date=2024-01-15&bins=30"
curl "http://localhost:8000/symbols"
```

And you get automatic OpenAPI docs at `http://localhost:8000/docs`.

## Node.js: The Web Frontend's Native Gateway

If your consumers are JavaScript applications, Node.js as the gateway makes the data pipeline feel native:

```javascript
// gateway.js
import express from 'express';
import { connect } from 'node-q';
import { promisify } from 'util';

const app = express();

// Create kdb+ connection pool
async function createConnection() {
  return new Promise((resolve, reject) => {
    const conn = connect({host: 'localhost', port: 5001});
    conn.on('connect', () => resolve(conn));
    conn.on('error', reject);
  });
}

let qConn;
(async () => {
  qConn = await createConnection();
  console.log('Connected to kdb+');
})();

function executeQ(query, ...args) {
  return new Promise((resolve, reject) => {
    qConn.k(query, ...args, (err, result) => {
      if (err) reject(err);
      else resolve(result);
    });
  });
}

app.get('/trades', async (req, res) => {
  const { sym, date } = req.query;
  if (!sym || !date) {
    return res.status(400).json({error: 'sym and date are required'});
  }

  try {
    const result = await executeQ(
      '{[s;d] select time, sym, price, size from trade where date=d, sym=s}',
      sym, new Date(date)
    );
    res.json(result);
  } catch (err) {
    res.status(500).json({error: err.message});
  }
});

app.listen(3000, () => console.log('Gateway on port 3000'));
```

## Choosing Your Approach

| Approach | Use when |
|----------|----------|
| q built-in HTTP | Simple dashboards, internal tooling, prototypes |
| FastAPI gateway | External APIs, need authentication/validation, Python team |
| Node.js gateway | Primary consumers are web apps, team is JS-native |
| Go gateway | High-throughput proxy, multiple downstream kdb+ processes |

The q built-in HTTP server is underrated for internal tooling — the zero-infrastructure story is real. For anything customer-facing or requiring proper auth, put a real web framework in front.
