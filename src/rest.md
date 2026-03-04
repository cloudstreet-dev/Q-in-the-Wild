# REST and Q: Building APIs Around kdb+/q

Most systems that need kdb+ data don't speak kdb+ IPC. They speak HTTP. Your data science team wants a REST endpoint they can call from pandas. Your mobile app team wants JSON. The compliance system wants a webhook. None of these are going to implement the Q wire protocol.

This chapter is about building REST APIs that sit in front of kdb+ — properly, with authentication, error handling, sensible data types, and the kind of response format that doesn't make your consumers write adapter code.

## The Architecture Decision

Three viable approaches, each with a distinct trade-off:

**1. Q's built-in HTTP (`.z.ph`/`.z.pp`)**
- Zero infrastructure: it's already running
- Good for internal tooling and simple cases
- Manual JSON serialization, no framework features
- Covered in the [Web Frameworks chapter](./webframeworks.md)

**2. External gateway (FastAPI, Flask, Go, Node.js)**
- Full web framework features: routing, middleware, validation, auth
- Proper OpenAPI/Swagger docs
- Separation of concerns: q does data, web framework does HTTP
- The right choice for external-facing APIs

**3. KX Insights / KX Dashboard (commercial)**
- KX's commercial API management layer
- Not covered here; see KX documentation if you have a license

We'll build a production-grade FastAPI gateway with authentication, OpenAPI docs, and proper error handling.

## Designing the API

Before writing code, decide on your conventions. These decisions are hard to change later:

**Date format**: ISO 8601 (`2024-01-15`), not Q's `2024.01.15`. REST APIs should be format-agnostic; Q dates are an implementation detail.

**Timestamps**: ISO 8601 with UTC timezone (`2024-01-15T09:30:00.123Z`). Not Q timestamps, not Unix milliseconds.

**Numbers**: JSON doesn't distinguish int from float. Document your precision guarantees. Prices are floats. Sizes are integers. Don't mix them up.

**Pagination**: Use `limit` and `offset` (or `cursor`-based for time-series). Don't return unbounded result sets.

**Errors**: Consistent error format: `{"error": "description", "code": "machine_readable_code"}`.

**Symbol naming**: Decide early whether API consumers pass `"AAPL"` or `"AAPL.O"` and stick with it. Your Q sym and the API sym may differ; the gateway is where you translate.

## A Production FastAPI Gateway

```python
# kdb_gateway/main.py
from fastapi import FastAPI, HTTPException, Depends, Query, Security
from fastapi.security.api_key import APIKeyHeader
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from contextlib import asynccontextmanager
from pydantic import BaseModel, Field
from typing import Optional
from datetime import date, datetime
import pykx as kx
import os
import logging

logger = logging.getLogger(__name__)

# ─── Configuration ──────────────────────────────────────────────────────────

KDB_HOST = os.getenv("KDB_HOST", "localhost")
KDB_PORT = int(os.getenv("KDB_PORT", "5001"))
KDB_USER = os.getenv("KDB_USER", "")
KDB_PASS = os.getenv("KDB_PASS", "")
API_KEYS = set(os.getenv("API_KEYS", "dev-key-123").split(","))

# ─── Connection Pool ──────────────────────────────────────────────────────────

class QPool:
    def __init__(self, host: str, port: int, user: str, password: str, size: int = 4):
        self._conns = []
        self._idx = 0
        self._host = host
        self._port = port
        self._user = user
        self._password = password
        self._size = size

    def start(self):
        for _ in range(self._size):
            auth = f"{self._user}:{self._password}" if self._user else ""
            conn = kx.SyncQConnection(
                host=self._host,
                port=self._port,
                username=self._user or None,
                password=self._password or None
            )
            self._conns.append(conn)
        logger.info(f"Connected to kdb+ at {self._host}:{self._port} ({self._size} connections)")

    def stop(self):
        for conn in self._conns:
            try:
                conn.close()
            except Exception:
                pass
        logger.info("kdb+ connections closed")

    def get(self) -> kx.SyncQConnection:
        conn = self._conns[self._idx % self._size]
        self._idx += 1
        return conn

    def query(self, q_code: str, *args):
        conn = self.get()
        try:
            if args:
                return conn(q_code, *args)
            return conn(q_code)
        except Exception as e:
            logger.error(f"kdb+ query error: {e}")
            raise

pool = QPool(KDB_HOST, KDB_PORT, KDB_USER, KDB_PASS)

# ─── App Lifecycle ──────────────────────────────────────────────────────────

@asynccontextmanager
async def lifespan(app: FastAPI):
    pool.start()
    yield
    pool.stop()

app = FastAPI(
    title="kdb+ Market Data API",
    description="REST API for accessing kdb+/q time-series market data",
    version="1.0.0",
    lifespan=lifespan
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# ─── Authentication ──────────────────────────────────────────────────────────

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

async def require_api_key(api_key: str = Security(api_key_header)):
    if api_key not in API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid or missing API key")
    return api_key

# ─── Response Models ──────────────────────────────────────────────────────────

class Trade(BaseModel):
    time: str
    sym: str
    price: float
    size: int

class OHLCBar(BaseModel):
    time: str
    open: float
    high: float
    low: float
    close: float
    volume: int
    vwap: float

class ErrorResponse(BaseModel):
    error: str
    code: str

# ─── Helpers ──────────────────────────────────────────────────────────────────

def serialize_result(result) -> list[dict]:
    """Convert pykx result to JSON-serializable list of dicts."""
    import pandas as pd
    import numpy as np

    df = result.pd()

    # Convert timestamps to ISO strings
    for col in df.columns:
        if df[col].dtype == 'datetime64[ns]':
            df[col] = df[col].dt.strftime('%Y-%m-%dT%H:%M:%S.%f')[:-3] + 'Z'
        elif df[col].dtype == 'object':
            df[col] = df[col].astype(str)

    return df.to_dict(orient="records")

def q_date(d: date) -> str:
    """Convert Python date to Q date literal."""
    return d.strftime("%Y.%m.%d")

# ─── Endpoints ───────────────────────────────────────────────────────────────

@app.get("/health")
async def health():
    """Health check. Tests kdb+ connectivity."""
    try:
        result = pool.query("1b")
        return {"status": "ok", "kdb_connected": True}
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "degraded", "kdb_connected": False, "error": str(e)}
        )

@app.get(
    "/trades",
    response_model=list[Trade],
    summary="Get trades for a symbol",
    responses={400: {"model": ErrorResponse}, 500: {"model": ErrorResponse}}
)
async def get_trades(
    sym: str = Query(..., description="Ticker symbol, e.g. AAPL", example="AAPL"),
    date: date = Query(..., description="Trading date"),
    limit: int = Query(1000, ge=1, le=10000, description="Maximum rows to return"),
    offset: int = Query(0, ge=0, description="Row offset for pagination"),
    _: str = Depends(require_api_key)
):
    """
    Retrieve trades for a symbol on a given date.

    Returns up to `limit` trades starting from `offset`, ordered by time.
    """
    try:
        result = pool.query(
            "{[s;d;n;o] select[n;o] time, sym, price, size from trade where date=d, sym=s}",
            kx.SymbolAtom(sym.upper()),
            kx.DateAtom(date),
            kx.LongAtom(limit),
            kx.LongAtom(offset)
        )
        return serialize_result(result)
    except kx.QError as e:
        raise HTTPException(status_code=400, detail={"error": str(e), "code": "q_error"})
    except Exception as e:
        logger.exception("Failed to query trades")
        raise HTTPException(status_code=500, detail={"error": "Internal error", "code": "internal"})

@app.get(
    "/ohlc",
    response_model=list[OHLCBar],
    summary="Get OHLC bars",
)
async def get_ohlc(
    sym: str = Query(..., description="Ticker symbol", example="AAPL"),
    date: date = Query(..., description="Trading date"),
    bars: int = Query(30, ge=1, le=390, description="Bar size in minutes"),
    _: str = Depends(require_api_key)
):
    """
    Retrieve OHLC (open/high/low/close) bars for a symbol.

    `bars` is the bar duration in minutes. 30 gives 30-minute bars,
    1 gives 1-minute bars (up to 390 for a full trading day).
    """
    try:
        result = pool.query(
            """{[s;d;b]
                select
                    open:   first price,
                    high:   max price,
                    low:    min price,
                    close:  last price,
                    volume: sum `long$size,
                    vwap:   size wavg price
                from trade
                where date=d, sym=s
                by time: b xbar time.minute
            }""",
            kx.SymbolAtom(sym.upper()),
            kx.DateAtom(date),
            kx.LongAtom(bars)
        )
        return serialize_result(result)
    except kx.QError as e:
        raise HTTPException(status_code=400, detail={"error": str(e), "code": "q_error"})
    except Exception as e:
        logger.exception("Failed to query OHLC")
        raise HTTPException(status_code=500, detail={"error": "Internal error", "code": "internal"})

@app.get("/symbols")
async def get_symbols(
    date: Optional[date] = Query(None, description="Date to check symbols for (default: today)"),
    _: str = Depends(require_api_key)
):
    """List available symbols, optionally filtered by date."""
    q_date_arg = date or __import__('datetime').date.today()
    try:
        result = pool.query(
            "{[d] exec distinct sym from trade where date=d}",
            kx.DateAtom(q_date_arg)
        )
        return {"symbols": sorted([str(s) for s in result.py()])}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/vwap")
async def get_vwap(
    date: date = Query(..., description="Trading date"),
    syms: Optional[str] = Query(None, description="Comma-separated symbols, e.g. AAPL,MSFT"),
    _: str = Depends(require_api_key)
):
    """
    Compute VWAP for all symbols (or a subset) on a date.

    Returns volume-weighted average price and total volume per symbol.
    """
    try:
        if syms:
            sym_list = [s.strip().upper() for s in syms.split(",")]
            result = pool.query(
                "{[d;ss] select vwap: size wavg price, volume: sum size by sym from trade where date=d, sym in ss}",
                kx.DateAtom(date),
                kx.SymbolVector(sym_list)
            )
        else:
            result = pool.query(
                "{[d] select vwap: size wavg price, volume: sum size by sym from trade where date=d}",
                kx.DateAtom(date)
            )
        return serialize_result(result)
    except kx.QError as e:
        raise HTTPException(status_code=400, detail={"error": str(e), "code": "q_error"})

# ─── POST: Execute Q (admin only) ────────────────────────────────────────────

class QQueryRequest(BaseModel):
    query: str = Field(..., description="Q expression to evaluate")

ADMIN_KEYS = set(os.getenv("ADMIN_API_KEYS", "admin-key-secret").split(","))

async def require_admin_key(api_key: str = Security(api_key_header)):
    if api_key not in ADMIN_KEYS:
        raise HTTPException(status_code=403, detail="Admin access required")
    return api_key

@app.post("/query")
async def execute_query(
    request: QQueryRequest,
    _: str = Depends(require_admin_key)
):
    """
    Execute an arbitrary Q query. Admin access only.

    Use this sparingly — prefer the typed endpoints above.
    """
    try:
        result = pool.query(request.query)
        df = result.pd()
        return {"result": df.to_dict(orient="records") if hasattr(df, 'to_dict') else str(result)}
    except kx.QError as e:
        raise HTTPException(status_code=400, detail={"error": str(e), "code": "q_error"})
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### Running It

```bash
pip install fastapi uvicorn pykx

# Development
uvicorn kdb_gateway.main:app --reload --port 8000

# Production
uvicorn kdb_gateway.main:app \
    --host 0.0.0.0 \
    --port 8000 \
    --workers 4 \
    --log-level info
```

### Environment Variables

```bash
export KDB_HOST=kdb.internal
export KDB_PORT=5001
export KDB_USER=apiuser
export KDB_PASS=secure_password
export API_KEYS="key1,key2,key3"
export ADMIN_API_KEYS="admin-secret"
```

### Testing

```bash
# Health check
curl http://localhost:8000/health

# Get trades (with API key)
curl -H "X-API-Key: dev-key-123" \
  "http://localhost:8000/trades?sym=AAPL&date=2024-01-15&limit=100"

# OHLC bars
curl -H "X-API-Key: dev-key-123" \
  "http://localhost:8000/ohlc?sym=AAPL&date=2024-01-15&bars=30"

# VWAP for multiple symbols
curl -H "X-API-Key: dev-key-123" \
  "http://localhost:8000/vwap?date=2024-01-15&syms=AAPL,MSFT,GOOG"

# OpenAPI docs (no auth needed)
open http://localhost:8000/docs
```

## Rate Limiting

Add rate limiting with `slowapi`:

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/trades")
@limiter.limit("100/minute")
async def get_trades(request: Request, ...):
    ...
```

## Caching

For expensive queries or endpoints where millisecond-fresh data isn't needed:

```python
from functools import lru_cache
import time

class TimedCache:
    def __init__(self, ttl_seconds: int = 60):
        self._cache = {}
        self._ttl = ttl_seconds

    def get(self, key: str):
        if key in self._cache:
            value, expires = self._cache[key]
            if time.time() < expires:
                return value
            del self._cache[key]
        return None

    def set(self, key: str, value):
        self._cache[key] = (value, time.time() + self._ttl)

symbol_cache = TimedCache(ttl_seconds=300)  # 5-minute cache for symbol list

@app.get("/symbols")
async def get_symbols(...):
    cache_key = f"symbols:{date}"
    cached = symbol_cache.get(cache_key)
    if cached:
        return cached

    result = pool.query(...)
    response = {"symbols": sorted([str(s) for s in result.py()])}
    symbol_cache.set(cache_key, response)
    return response
```

## Docker Deployment

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "kdb_gateway.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
services:
  kdb-gateway:
    build: .
    ports:
      - "8000:8000"
    environment:
      KDB_HOST: kdb
      KDB_PORT: 5001
      API_KEYS: "${API_KEYS}"
    depends_on:
      - kdb
    restart: unless-stopped

  kdb:
    image: kxsys/kdb:latest
    ports:
      - "5001:5001"
    command: ["-p", "5001"]
```

## OpenAPI Integration

FastAPI generates OpenAPI (Swagger) docs automatically. At `http://localhost:8000/docs` you get an interactive API explorer, and at `http://localhost:8000/openapi.json` you get the schema for generating client libraries:

```bash
# Generate a Python client from the OpenAPI spec
pip install openapi-generator-cli
openapi-generator-cli generate \
    -i http://localhost:8000/openapi.json \
    -g python \
    -o ./generated-client \
    --package-name kdb_client
```

Your consumers get a typed client without writing any integration code. That's the kind of thing that makes the Python team happy with you.
