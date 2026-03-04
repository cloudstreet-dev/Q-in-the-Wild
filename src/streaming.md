# Streaming Integrations: Kafka, WebSockets, and Real-Time Q

kdb+/q was built for real-time data. The tick infrastructure — the tickerplant, real-time database, and historical database — is one of Q's strongest features and the reason it dominates in high-frequency finance. What it wasn't originally designed for is plugging into the broader streaming ecosystem: Kafka topics, WebSocket-hungry frontends, and the event-driven microservices architecture that everyone built in the 2020s.

This chapter is about making Q a first-class participant in that ecosystem — both as a consumer of external streams and as a producer of real-time data.

## Q's Native Tick Architecture (Brief Review)

Before plugging Q into Kafka, understand what Q already does natively:

```
Feed Handler → Tickerplant → RDB (real-time DB)
                           → Subscribers (other q processes)
                           → Log file → HDB (historical DB)
```

The tickerplant receives data, timestamps it, writes to a log, and publishes to subscribers via IPC. Each subscriber calls `.u.sub` to register and receives updates via `.u.upd`.

This is already a streaming architecture — it's just Q-native. When you need to integrate with non-Q systems, you extend this pattern rather than replacing it.

## Kafka Integration

### Architecture

The two Kafka-Q integration patterns:

**Q as consumer**: A Q process subscribes to Kafka topics and ingests data into kdb+ tables. Useful for bringing external event streams into your time-series database.

**Q as producer**: A Q process (typically a tickerplant subscriber) publishes processed data to Kafka topics for consumption by other services.

### KafkaQ

The `KafkaQ` library (from KX) provides Kafka integration for q:

```bash
# Install KafkaQ (requires librdkafka)
# macOS
brew install librdkafka
# Ubuntu/Debian
apt-get install librdkafka-dev

# Download KafkaQ
# https://github.com/KxSystems/kafka
# Place kafkaq.so/.dylib in your q path
```

```q
/ Load KafkaQ
\l kafkaq.q

/ ─── Consumer ───────────────────────────────────────────────────────────────

/ Create a consumer with configuration
consumer: .kafka.newConsumer[
    `bootstrap.servers`group.id`auto.offset.reset!
    ("localhost:9092"; "my-q-consumer"; "earliest")
]

/ Subscribe to topics
.kafka.subscribe[consumer; `trade_data`order_data]

/ Message handler — called for each incoming Kafka message
.kafka.recvMsg: {[consumer; msg]
    / msg is a dictionary: `topic`partition`offset`key`data`timestamp
    topic: msg`topic;
    payload: .j.k msg`data;   / deserialize JSON payload

    / Route to appropriate handler
    $[topic ~ "trade_data";   ingestTrade payload;
      topic ~ "order_data";   ingestOrder payload;
      / default
      -1 "Unknown topic: ", string topic]
    }

/ Start polling (call in a timer or loop)
.kafka.poll[consumer; 1000]   / poll with 1000ms timeout

/ ─── Trade ingestion ─────────────────────────────────────────────────────────

/ In-memory trade table (replace with actual schema)
trade: ([] time:`timestamp$(); sym:`symbol$(); price:`float$(); size:`long$())

ingestTrade: {[payload]
    / Convert Kafka message to Q row
    row: (
        `timestamp$payload`timestamp;
        `$payload`sym;
        `float$payload`price;
        `long$payload`size
    );
    `trade insert row;

    / Optional: forward to tickerplant
    if[0 < count tp_handle; neg[tp_handle] (`.u.upd; `trade; row)]
    }
```

### Q as Kafka Producer

Publishing Q data to Kafka:

```q
\l kafkaq.q

/ Create a producer
producer: .kafka.newProducer[
    `bootstrap.servers`acks!
    ("localhost:9092"; "all")
]

/ Publish tickerplant updates to Kafka
/ Add this to your RDB subscriber

.u.upd: {[t; data]
    / Standard RDB upsert
    t insert data;

    / Also publish to Kafka
    publishToKafka[t; data]
    }

publishToKafka: {[table; data]
    topic: "kdb-", string table;

    / Convert each row to JSON and publish
    rows: flip data;
    do[count rows 0;
        row: rows[; i];
        msg: .j.j `table`data!(table; row);
        .kafka.publish[producer; `$topic; ""; msg]
    ]
    }
```

### Python Kafka Bridge

If installing KafkaQ feels like more than you want to manage, a Python bridge is a clean alternative:

```python
# kafka_bridge.py
# Consumes from Kafka, inserts into kdb+ via IPC

from kafka import KafkaConsumer
import pykx as kx
import json
import logging
from datetime import datetime

logger = logging.getLogger(__name__)

def run_bridge(
    kafka_brokers: str,
    kafka_topic: str,
    kdb_host: str,
    kdb_port: int,
    kdb_table: str
):
    consumer = KafkaConsumer(
        kafka_topic,
        bootstrap_servers=kafka_brokers,
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        auto_offset_reset='latest',
        group_id='kdb-bridge'
    )

    q = kx.SyncQConnection(host=kdb_host, port=kdb_port)
    logger.info(f"Bridge running: {kafka_topic} → kdb+:{kdb_port}:{kdb_table}")

    batch = []
    batch_size = 100

    try:
        for message in consumer:
            payload = message.value
            batch.append(payload)

            if len(batch) >= batch_size:
                flush_batch(q, kdb_table, batch)
                batch = []

    except KeyboardInterrupt:
        if batch:
            flush_batch(q, kdb_table, batch)
    finally:
        consumer.close()
        q.close()

def flush_batch(q: kx.SyncQConnection, table: str, batch: list):
    """Insert a batch of records into kdb+."""
    import pandas as pd

    df = pd.DataFrame(batch)

    # Type coercions
    if 'timestamp' in df.columns:
        df['timestamp'] = pd.to_datetime(df['timestamp'])
    if 'sym' in df.columns:
        df['sym'] = df['sym'].astype(str)

    q(f'`{table} insert', df)
    logger.info(f"Flushed {len(batch)} records to {table}")

if __name__ == '__main__':
    run_bridge(
        kafka_brokers='localhost:9092',
        kafka_topic='market-trades',
        kdb_host='localhost',
        kdb_port=5001,
        kdb_table='trade'
    )
```

## WebSockets: Streaming Q Data to Browsers

The use case: a dashboard that shows live market data, updated as trades arrive in kdb+. WebSockets are the right transport for this — persistent connection, low overhead, bidirectional.

### Q's Built-In WebSocket Server

kdb+ 3.5+ has native WebSocket support via `.z.wo`, `.z.wc`, `.z.ws`:

```q
\p 5001   / start on port 5001 (same port handles HTTP, IPC, and WebSockets)

/ WebSocket connection tracking
ws_clients: `long$()   / list of connected WebSocket handles

.z.wo: {[h]
    / h = new WebSocket handle
    ws_clients,: h;
    -1 "WS connected: ", string h;
    }

.z.wc: {[h]
    / h = closing WebSocket handle
    ws_clients: ws_clients except h;
    -1 "WS disconnected: ", string h;
    }

.z.ws: {[x]
    / x = incoming WebSocket message (string or bytes)
    / Parse request and send back data
    request: .j.k x;

    response: handleWsRequest request;
    neg[.z.w] .j.j response
    }

handleWsRequest: {[req]
    action: req`action;
    $[action ~ "subscribe";
        handleSubscribe req;
      action ~ "query";
        handleQuery req;
      `error`message!("unknown_action"; action)]
    }

handleSubscribe: {[req]
    syms: `$req`syms;
    / Store subscription: handle -> syms
    `subscriptions upsert (.z.w; syms);
    `status`message!("subscribed"; "Subscribed to ", " " sv string syms)
    }

handleQuery: {[req]
    q_code: req`query;
    result: @[value; q_code; {[e] `error`message!(1b; string e)}];
    `status`data!("ok"; result)
    }

/ Broadcast trade updates to subscribed WebSocket clients
subscriptions: ([handle:`long$()] syms:())

broadcastTrade: {[trade_data]
    / For each connected WS client with matching subscription
    {[h; trade_data]
        client_syms: subscriptions[h; `syms];
        matching: trade_data where trade_data[`sym] in client_syms;
        if[count matching;
            neg[h] .j.j `type`data!("trade_update"; matching)]
    }[; trade_data] each ws_clients;
    }

/ Hook into tickerplant update function
.u.upd: {[t; data]
    $[t ~ `trade;
        broadcastTrade data;
        ()];
    }
```

### JavaScript Client

```javascript
// dashboard.js
const ws = new WebSocket('ws://localhost:5001');

ws.onopen = () => {
    console.log('Connected to kdb+');

    // Subscribe to specific symbols
    ws.send(JSON.stringify({
        action: 'subscribe',
        syms: ['AAPL', 'MSFT', 'GOOG']
    }));
};

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);

    if (msg.type === 'trade_update') {
        updateDashboard(msg.data);
    }
};

ws.onerror = (error) => console.error('WebSocket error:', error);
ws.onclose = () => {
    console.log('Disconnected');
    setTimeout(() => reconnect(), 5000);  // auto-reconnect
};

function updateDashboard(trades) {
    trades.forEach(trade => {
        const el = document.getElementById(`price-${trade.sym}`);
        if (el) {
            el.textContent = trade.price.toFixed(2);
            el.classList.add('flash');
            setTimeout(() => el.classList.remove('flash'), 300);
        }
    });
}

// Request historical data via query
function fetchOHLC(sym, date) {
    ws.send(JSON.stringify({
        action: 'query',
        query: `select from ohlc where sym=\`${sym}, date=${date}`
    }));
}
```

### FastAPI WebSocket Proxy

For production, a FastAPI proxy handles authentication, connection management, and adds observability:

```python
# websocket_proxy.py
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from fastapi.websockets import WebSocketState
import pykx as kx
import asyncio
import json
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

# kdb+ connection for WebSocket data
q = kx.SyncQConnection(host='localhost', port=5001)

class ConnectionManager:
    def __init__(self):
        self.active: dict[str, list[WebSocket]] = {}  # sym -> [websocket]

    async def connect(self, websocket: WebSocket, sym: str):
        await websocket.accept()
        if sym not in self.active:
            self.active[sym] = []
        self.active[sym].append(websocket)
        logger.info(f"WS connected for {sym}, total: {sum(len(v) for v in self.active.values())}")

    def disconnect(self, websocket: WebSocket, sym: str):
        if sym in self.active:
            self.active[sym] = [ws for ws in self.active[sym] if ws != websocket]

    async def broadcast(self, sym: str, data: dict):
        if sym not in self.active:
            return
        dead = []
        for ws in self.active[sym]:
            try:
                if ws.client_state == WebSocketState.CONNECTED:
                    await ws.send_json(data)
            except Exception:
                dead.append(ws)
        for ws in dead:
            self.active[sym].remove(ws)

manager = ConnectionManager()

@app.websocket("/stream/{sym}")
async def stream_symbol(websocket: WebSocket, sym: str):
    """Stream real-time trades for a symbol."""
    await manager.connect(websocket, sym.upper())
    sym = sym.upper()

    try:
        # Send initial snapshot
        snapshot = q(
            "{[s] select[-50] time, price, size from trade where date=.z.d, sym=s}",
            kx.SymbolAtom(sym)
        ).pd()
        await websocket.send_json({
            "type": "snapshot",
            "sym": sym,
            "data": snapshot.to_dict(orient="records")
        })

        # Keep connection alive, receive control messages
        while True:
            try:
                msg = await asyncio.wait_for(websocket.receive_json(), timeout=30)
                if msg.get("action") == "ping":
                    await websocket.send_json({"type": "pong"})
            except asyncio.TimeoutError:
                # Send keepalive
                await websocket.send_json({"type": "heartbeat"})

    except WebSocketDisconnect:
        manager.disconnect(websocket, sym)
        logger.info(f"WS disconnected: {sym}")

# Background task: poll kdb+ and broadcast updates
async def poll_and_broadcast():
    """Poll kdb+ for new trades and broadcast to subscribers."""
    last_time = {}  # sym -> last seen time

    while True:
        await asyncio.sleep(0.5)  # 500ms polling interval

        for sym in list(manager.active.keys()):
            if not manager.active.get(sym):
                continue
            try:
                cutoff = last_time.get(sym, "00:00:00.000")
                new_trades = q(
                    "{[s;t] select time, price, size from trade where date=.z.d, sym=s, time>t}",
                    kx.SymbolAtom(sym),
                    kx.TimeAtom(cutoff)
                ).pd()

                if len(new_trades) > 0:
                    last_time[sym] = str(new_trades['time'].max())
                    await manager.broadcast(sym, {
                        "type": "update",
                        "sym": sym,
                        "trades": new_trades.to_dict(orient="records")
                    })
            except Exception as e:
                logger.error(f"Poll error for {sym}: {e}")

@app.on_event("startup")
async def startup():
    asyncio.create_task(poll_and_broadcast())
```

## Handling Back-Pressure

When Q is producing data faster than consumers can handle it, you need a back-pressure strategy:

```q
/ Simple back-pressure in Q: check send buffer before sending
broadcastWithBackPressure: {[h; data]
    / .z.W[h] = bytes in send buffer for handle h
    buffer_size: .z.W[h];

    $[buffer_size > 1000000;   / 1MB threshold
        / Buffer full: drop or queue
        (-1 "Dropping update for slow consumer: ", string h);
        / Buffer OK: send
        neg[h] .j.j data]
    }
```

For Kafka as a buffer (the right architectural answer for high-throughput):

```
kdb+ tickerplant
    → publishes to Kafka topic (Q→Kafka producer)
    → Kafka topic acts as buffer
    → slow consumers read from Kafka at their own pace
```

This decouples the producer (kdb+) from the consumer's throughput.

## The Complete Real-Time Pipeline

Putting it together: Kafka → Q → WebSocket → Browser:

```
[Market Data Feed]
        ↓
[Kafka Topic: raw-trades]
        ↓
[kdb+ Feed Handler] → [Tickerplant] → [RDB]
                                     ↓
                            [WebSocket Server]
                                     ↓
                         [Browser Dashboard]
                                     ↓
                              [REST API]
                                     ↓
                         [External Consumers]
```

Each layer has a clear responsibility:
- **Kafka**: durable message queue, decouples feed from processing
- **kdb+ tickerplant**: time-stamping, logging, fan-out
- **RDB**: in-memory storage for today's data
- **WebSocket server**: real-time push to browsers
- **REST API**: request-response for historical data

Q sits in the middle of this, doing what it does best: storing and querying time-series data at speed.
