# Python and Q: PyQ, qPython, and Getting Along

There are two serious Python-Q integration libraries, and they have different philosophies, different trade-offs, and different ways of making you feel like you chose wrong. Let's save you the afternoon of discovery.

**PyQ**: KX's official library. Deep integration — runs Python inside the q process, shares memory, no serialization overhead for in-process calls. Requires a real KX installation. Heavyweight but serious.

**qPython**: Pure Python, pure IPC. Connects to a running kdb+ process over the network. No special installation, no KX runtime dependency. The right choice for most external consumers of kdb+ data.

**pykx**: KX's newer, actively-developed library. Wraps the KX C API, can run embedded or in IPC mode, has a pandas-compatible API, and is where KX is putting its energy. If you're starting fresh, this is the one.

We'll cover all three, but spend the most time on qPython (because you probably already have a kdb+ process and just want to talk to it) and pykx (because it's the future).

## qPython: The Pragmatic Choice

### Installation

```bash
pip install qpython
```

That's it. No KX runtime. Works against any kdb+ 3.x or 4.x process.

### Connecting and Basic Queries

```python
from qpython import qconnection
from qpython.qtype import QException

# Connect to a running kdb+ process
q = qconnection.QConnection(host='localhost', port=5001)
q.open()

# Execute a simple expression
result = q('2 + 2')
print(result)  # 4

# Execute a function call
result = q('til 10')
print(result)  # [0 1 2 3 4 5 6 7 8 9]

q.close()
```

Use it as a context manager to avoid forgetting to close:

```python
from qpython import qconnection

with qconnection.QConnection(host='localhost', port=5001) as q:
    result = q('select from trade where date=2024.01.15, sym=`AAPL')
    print(result)
```

### Type Mapping: Where Things Get Interesting

Q's type system is richer than Python's. When qPython receives data, it maps Q types to numpy arrays and pandas objects. Understanding this mapping saves debugging sessions.

```python
import numpy as np
from qpython import qconnection
from qpython.qtype import (
    QSymbolList, QTimestampList, QFloat64List, QLONG_LIST
)

with qconnection.QConnection(host='localhost', port=5001, pandas=True) as q:
    # With pandas=True, tables come back as DataFrames
    df = q('select time, sym, price, size from trade where date=2024.01.15')
    print(df.dtypes)
    # time     datetime64[ns]
    # sym      object          <- Q symbols become Python strings
    # price    float64
    # size     int64

    print(df.head())
```

The `pandas=True` flag is almost always what you want for tabular data. Without it, you get a QTable object that you then have to convert anyway.

### Sending Data to Q

Pushing data from Python to Q is where type handling gets tricky. Q is strict about types; Python is not.

```python
import numpy as np
import pandas as pd
from qpython import qconnection
from qpython.qtype import QSymbolList, QLONG, QFLOAT64

with qconnection.QConnection(host='localhost', port=5001, pandas=True) as q:
    # Create a DataFrame to push
    df = pd.DataFrame({
        'time': pd.to_datetime(['2024-01-15 09:30:00', '2024-01-15 09:30:01']),
        'sym': ['AAPL', 'MSFT'],
        'price': [185.50, 375.20],
        'size': [100, 200]
    })

    # Push a variable into Q's namespace
    q('set', np.string_('mydata'), df)

    # Verify it's there
    result = q('count mydata')
    print(result)  # 2

    # Now query it
    result = q('select from mydata where sym=`AAPL')
    print(result)
```

### The Symbol Problem

Q symbols and Python strings are conceptually similar but mechanically different. When you query Q and get back symbol data, qPython gives you Python bytes objects by default (without `pandas=True`) or strings (with it). When you *send* symbols to Q, you need to be explicit:

```python
from qpython.qtype import QSymbolList
import numpy as np

with qconnection.QConnection(host='localhost', port=5001) as q:
    # Wrong: Python list of strings won't be interpreted as Q symbols
    # q('insert', np.string_('mytable'), [['AAPL', 'MSFT'], [100, 200]])

    # Right: use QSymbolList for symbol columns
    syms = QSymbolList(np.array(['AAPL', 'MSFT'], dtype='S'))
    sizes = np.array([100, 200], dtype=np.int64)

    q('`mytable insert', (syms, sizes))
```

The difference between bytes and symbols matters in Q. If `sym` in your trade table is a symbol and you send bytes, you'll get a type error on join operations that will be deeply confusing for ten minutes.

### Asynchronous Calls

For fire-and-forget operations (publishing to a tickerplant, for instance):

```python
from qpython import qconnection

with qconnection.QConnection(host='localhost', port=5001) as q:
    # Synchronous — waits for response
    result = q('.u.upd', np.string_('trade'), data)

    # Asynchronous — returns immediately
    q('.u.upd', np.string_('trade'), data, sync=False)
```

Be careful with async calls in tight loops — you can flood a Q process faster than it can handle messages. Add a small sleep or batch your updates if you're publishing high-frequency data.

### Error Handling

Q errors propagate back as `QException`:

```python
from qpython.qtype import QException

with qconnection.QConnection(host='localhost', port=5001) as q:
    try:
        result = q('1 + `sym')  # type error
    except QException as e:
        print(f"Q error: {e}")  # type
    except Exception as e:
        print(f"Connection error: {e}")
```

## pykx: The Modern Choice

pykx is KX's actively-maintained library that has both an embedded mode (runs kdb+ in-process) and an IPC mode (connects to a running process). The API is cleaner than qPython and it has better pandas and numpy integration.

### Installation

```bash
pip install pykx
```

For IPC-only mode (no KX license needed):

```bash
# Set this before importing pykx to use IPC-only mode
export PYKX_LICENSED_LIBRARIES=false
# Or in Python:
import os
os.environ['PYKX_LICENSED_LIBRARIES'] = 'false'
```

### IPC Mode

```python
import pykx as kx

# Connect to a running kdb+ process
with kx.SyncQConnection(host='localhost', port=5001) as q:
    # Execute q code
    result = q('select from trade where date=2024.01.15')

    # Convert to pandas
    df = result.pd()
    print(df)

    # Or arrow
    table = result.pa()
```

### pykx Type System

pykx has a clean type hierarchy that mirrors Q's:

```python
import pykx as kx

with kx.SyncQConnection(host='localhost', port=5001) as q:
    # Atoms
    x = q('42')
    print(type(x))         # pykx.LongAtom
    print(x.py())          # 42 (Python int)

    # Lists
    xs = q('1 2 3f')
    print(type(xs))        # pykx.FloatVector
    print(xs.np())         # numpy array([1., 2., 3.])

    # Tables come back as pykx.Table
    tbl = q('([] a:1 2 3; b:`x`y`z)')
    print(tbl.pd())        # pandas DataFrame
```

### Sending Typed Data

```python
import pykx as kx
import pandas as pd
import numpy as np

with kx.SyncQConnection(host='localhost', port=5001) as q:
    # pykx can convert pandas DataFrames directly
    df = pd.DataFrame({
        'sym': ['AAPL', 'MSFT', 'GOOG'],
        'price': [185.5, 375.2, 140.8],
        'size': [100, 200, 150]
    })

    # Set a variable in Q
    q['mydata'] = df

    # Call a Q function with a Python argument
    result = q('{[t] select from t where price > 200}', df)
    print(result.pd())
```

### Async Connections

```python
import pykx as kx
import asyncio

async def publish_data():
    async with kx.AsyncQConnection(host='localhost', port=5001) as q:
        # Async query
        result = await q('select from trade')
        df = result.pd()

        # Fire-and-forget publish
        await q('.u.upd', 'trade', df, wait=False)
```

## PyQ: The Deep Integration Option

PyQ is for when you want Python to run *inside* the q process — sharing memory, zero serialization overhead, using Q data structures directly from Python. It's powerful and it requires a full KX installation.

```bash
pip install pyq
# Run with q's Python interpreter:
pyq my_script.py
```

Inside a PyQ session:

```python
from pyq import q, K

# Q functions are accessible as attributes
q.til(10)            # calls q's til function
q('.', 'mylib.q')   # loads a Q script

# K objects are Q values
x = K(3.14)
y = q.sqrt(x)

# Create Q tables from Python
import numpy as np
data = {
    'price': np.array([100.5, 101.2, 99.8]),
    'size': np.array([100, 200, 150], dtype=np.int64)
}
q['mytrade'] = data
```

PyQ's strength is in embedded analytics — Python ML libraries operating on Q data without copying it. If you're running TensorFlow or scikit-learn against live kdb+ data and latency matters, PyQ is the right architecture.

## Which One Should You Use?

| Scenario | Library |
|----------|---------|
| External service consuming kdb+ data | qPython or pykx IPC |
| New project, greenfield setup | pykx |
| High-frequency publisher to tickerplant | qPython (async) or pykx |
| ML/analytics running inside q process | PyQ |
| Don't have KX license, just want data | qPython |
| Need pandas/arrow integration | pykx |

The honest answer: **start with pykx**. Its API is cleaner, it's actively maintained, and it handles the type system better than qPython. Use qPython if you need a lighter dependency or you're working with legacy code that already uses it. Use PyQ only if you genuinely need in-process Python execution.

## A Complete Example: Time-Series Analysis Pipeline

This is the pattern that actually comes up in production — Python consuming kdb+ data for analysis, then writing results back.

```python
import pykx as kx
import pandas as pd
import numpy as np
from scipy import stats

def compute_vwap(df: pd.DataFrame) -> pd.DataFrame:
    """Compute VWAP from trade data."""
    df['notional'] = df['price'] * df['size']
    vwap = df.groupby('sym').apply(
        lambda x: x['notional'].sum() / x['size'].sum()
    ).reset_index()
    vwap.columns = ['sym', 'vwap']
    return vwap

def compute_volatility(df: pd.DataFrame, window: int = 20) -> pd.DataFrame:
    """Rolling annualized volatility."""
    df = df.sort_values(['sym', 'time'])
    df['log_ret'] = df.groupby('sym')['price'].transform(
        lambda x: np.log(x / x.shift(1))
    )
    df['vol'] = df.groupby('sym')['log_ret'].transform(
        lambda x: x.rolling(window).std() * np.sqrt(252)
    )
    return df[['time', 'sym', 'vol']].dropna()

def main():
    with kx.SyncQConnection(host='localhost', port=5001) as q:
        # Pull today's trade data
        trades = q('''
            select time, sym, price, size
            from trade
            where date=.z.d,
                  sym in `AAPL`MSFT`GOOG`AMZN`META
        ''').pd()

        print(f"Loaded {len(trades):,} trades")

        # Compute analytics in Python
        vwap = compute_vwap(trades)
        vol = compute_volatility(trades)

        # Write results back to Q
        q['analytics_vwap'] = vwap
        q['analytics_vol'] = vol

        # Or insert into a Q table
        q('{`results_vwap upsert x}', vwap)

        print("Analytics written to Q")
        print(vwap)

if __name__ == '__main__':
    main()
```

## Timestamp Handling (The Thing That Will Burn You Once)

Q timestamps are nanoseconds since 2000.01.01. Python datetimes are microsecond-resolution since 1970-01-01. Every library does the conversion for you, but sometimes it goes wrong.

```python
import pykx as kx
import pandas as pd

with kx.SyncQConnection(host='localhost', port=5001) as q:
    # Q timestamp
    ts = q('2024.01.15D09:30:00.123456789')

    # pykx converts correctly
    print(ts.py())  # datetime.datetime(2024, 1, 15, 9, 30, 0, 123456)
                    # Note: microseconds, nanoseconds truncated

    # For nanosecond precision, use numpy
    print(ts.np())  # numpy.datetime64('2024-01-15T09:30:00.123456789')

    # pandas preserves nanoseconds
    df = q('([] t:enlist 2024.01.15D09:30:00.123456789)').pd()
    print(df['t'].dtype)      # datetime64[ns]
    print(df['t'].iloc[0])    # 2024-01-15 09:30:00.123456789
```

The gotcha: if you use `ts.py()` on a timestamp column, you lose the nanoseconds silently. Use `.np()` or work through pandas for nanosecond-resolution data.

## Authentication

Both libraries support authenticated connections:

```python
# qPython
q = qconnection.QConnection(
    host='localhost',
    port=5001,
    username='myuser',
    password='mypass'
)

# pykx
q = kx.SyncQConnection(
    host='localhost',
    port=5001,
    username='myuser',
    password='mypass'
)
```

Q's authentication is username:password in the connection handshake. If the server has `.z.pw` defined, it uses that for validation; otherwise it uses the OS password file. For production, use `.z.pw` with proper credential management — don't hardcode passwords in code that's going into version control.
