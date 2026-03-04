# Developer Tooling: Linters, Formatters, and Making Q Less Lonely

Q development has historically been a solitary activity. You write code in a text editor, paste it into a REPL, and find out what's wrong when it errors. There's no static type checker, the formatter options are limited, and "test suite" in many Q shops means "I ran it and it seemed fine."

This has improved. The improvement is modest compared to what Python, Rust, or JavaScript developers take for granted, but it's real and worth using.

## Linting: qls and the KX Language Server

The **KX Language Server** (`qls`) is the most capable static analysis tool available for Q. It powers the VS Code extension's diagnostics and can be run standalone for CI integration.

What it checks:
- Undefined variables (within analyzable scope)
- Type errors it can infer statically
- Unused variable assignments
- Shadowed names
- Wrong number of arguments to known functions
- Some style issues

### Using qls in VS Code

The KX VS Code extension installs and runs `qls` automatically. Underlined code in the editor with a red/yellow squiggle is `qls` talking to you. Take it seriously.

Common diagnostics:

```q
/ qls will flag this:
myFunc: {[x; y]
    result: x + z   / 'z' is not defined in this scope
    result * y
    }

/ And this:
unused: 42   / assigned but never used in the function

/ And this (wrong arg count):
count[1; 2]   / count takes 1 argument

/ This is fine:
f: {[x] x * 2}
g: {[x] f x}
```

### qls in CI

For automated code quality checks:

```bash
# Install (part of the KX VS Code extension bundle, also available standalone)
npm install -g @kxsystems/kdb-lsp

# Run on Q files
qls --check src/**/*.q

# With specific rules
qls --check --rules=all src/

# JSON output for programmatic processing
qls --check --format=json src/*.q | jq '.diagnostics[] | select(.severity == "error")'
```

GitHub Actions integration:

```yaml
# .github/workflows/lint.yml
name: Lint Q Code

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install kdb-lsp
        run: npm install -g @kxsystems/kdb-lsp

      - name: Lint Q files
        run: |
          qls --check --format=json q/**/*.q > lint_results.json || true
          cat lint_results.json
          # Fail if any errors
          errors=$(cat lint_results.json | jq '[.diagnostics[] | select(.severity == "error")] | length')
          if [ "$errors" -gt "0" ]; then
            echo "Found $errors lint errors"
            exit 1
          fi
```

## Formatting: The State of Q Formatting

Q has no official formatter equivalent to `gofmt`, `rustfmt`, or `black`. This is a gap in the tooling ecosystem.

What exists:
- `qformat` (community): A Python-based formatter for Q code. Not as opinionated as `black`, but handles basic whitespace and indentation.
- Manual conventions: Most Q shops define their own style guide and enforce it in code review.

Install and use `qformat`:

```bash
pip install qformat

# Format in place
qformat --in-place mycode.q

# Check without modifying (for CI)
qformat --check mycode.q && echo "Formatted" || echo "Needs formatting"
```

### Q Style Conventions (The Practical Guide)

Since there's no `gofmt` that enforces these for you, here are conventions worth adopting:

```q
/ ─── Naming ──────────────────────────────────────────────────────────────────

/ Functions: camelCase
computeVwap: {[trades] ...}

/ Tables: lowercase
trade: ([] time:`timestamp$(); sym:`symbol$(); price:`float$())

/ Namespaces: lowercase with dot
.myapp.config: `host`port!("localhost"; 5001)
.myapp.init: {[] ...}

/ Constants: UPPERCASE (informal convention)
MAX_SIZE: 1000000

/ ─── Function Style ──────────────────────────────────────────────────────────

/ Named arguments for multi-arg functions (don't use x y z for non-trivial functions)
computeOHLC: {[trades; sym; date; barMins]
    select
        open:  first price,
        high:  max price,
        low:   min price,
        close: last price
    from trades
    where date=date, sym=sym
    by time: barMins xbar time.minute
    }

/ Short lambda: x y z are fine for 1-3 arg lambdas with obvious semantics
doubled: {x * 2}
sumPair: {x + y}

/ ─── Spacing ─────────────────────────────────────────────────────────────────

/ Space around operators in expressions
a: b + c * d     / good
a:b+c*d          / bad (hard to read at speed)

/ No space before colon in assignment (the q way)
result: 42       / good
result : 42      / bad

/ Space after semicolon in multi-statement function
f: {[x] a: x+1; b: a*2; b-1}  / acceptable for short functions

/ Multi-line for complex functions
f: {[x]
    a: x + 1;
    b: a * 2;
    b - 1
    }

/ ─── Documentation ──────────────────────────────────────────────────────────

/ Use line comments for non-obvious logic
/ Use block comments for function documentation

/ Block comment style for function docs:
/ computeVwap: Compute volume-weighted average price
/   trades: trade table with columns time, sym, price, size
/   sym: symbol to compute VWAP for
/   date: date to compute VWAP for
/   returns: float (VWAP) or null if no trades
computeVwap: {[trades; sym; date]
    data: select from trades where date=date, sym=sym;
    $[count data;
        data[`size] wavg data[`price];
        0n]
    }
```

## Testing: q-unit and Homemade Test Frameworks

Q doesn't have a built-in test framework. The options are:

**`q-unit`**: A minimal test framework for Q. Similar in spirit to xUnit.

**`qspec`**: A BDD-style testing framework. Less used but cleaner for descriptive tests.

**Roll your own**: Many Q shops use a simple convention file that tests functions inline.

### q-unit

```bash
# Download q-unit
# https://github.com/CharlesSkelton/qunit
# Place qunit.q in your q path
```

```q
/ test_trade.q
\l qunit.q

/ Test: VWAP calculation
test_vwap_basic: {
    trades: ([] price: 100.0 101.0 99.0; size: 100 200 150);
    expected: (100.0*100 + 101.0*200 + 99.0*150) % (100+200+150);
    result: computeVwap trades;
    .qunit.assertEqual[result; expected; "basic VWAP"]
    }

test_vwap_empty: {
    trades: ([] price:`float$(); size:`long$());
    result: computeVwap trades;
    .qunit.assertNull[result; "empty input returns null"]
    }

test_vwap_single: {
    trades: ([] price: enlist 100.0; size: enlist 100);
    result: computeVwap trades;
    .qunit.assertEqual[result; 100.0; "single trade VWAP"]
    }

/ Run tests
.qunit.run[]
```

### Simple Homemade Framework

If adding a dependency feels heavy, this pattern works fine for smaller codebases:

```q
/ test.q — minimal test runner

.test.passed: 0;
.test.failed: 0;

/ assert: check a condition
.test.assert: {[name; condition]
    $[condition;
        [.test.passed +: 1;
         -1 "[PASS] ", name];
        [.test.failed +: 1;
         -1 "[FAIL] ", name]]
    }

/ assertEqual: check equality
.test.assertEqual: {[name; actual; expected]
    .test.assert[name; actual ~ expected]
    }

/ assertError: check that an expression throws
.test.assertError: {[name; expr]
    result: @[value; expr; {[e] 1b}];
    .test.assert[name; result ~ 1b]
    }

/ run: execute test function, report results
.test.run: {[testFns]
    testFns@\: ();
    -1 "\nResults: ", (string .test.passed), " passed, ",
       (string .test.failed), " failed";
    if[.test.failed > 0; exit 1]   / non-zero exit for CI
    }

/ ─── Example tests ──────────────────────────────────────────────────────────

testComputeVwap: {[]
    .test.assertEqual[
        "basic VWAP";
        computeVwap ([] price:100.0 101.0; size:100 200);
        (100.0*100 + 101.0*200) % 300
    ];
    .test.assertError[
        "null sym throws";
        {computeVwap ([] price:1.0; size:1); (0N; .z.d)}
    ]
    }

testScalePrice: {[]
    .test.assertEqual["scale by 2"; scalePrice[2.0; 100.0]; 200.0];
    .test.assertEqual["scale by 0"; scalePrice[0.0; 100.0]; 0.0];
    .test.assertEqual["negative scale"; scalePrice[-1.0; 100.0]; -100.0]
    }

.test.run `testComputeVwap`testScalePrice
```

Run from the command line:
```bash
q test.q -q   / -q for quiet mode (suppresses startup output)
echo "Exit code: $?"  / 0 = all tests pass, 1 = failures
```

### CI Integration for Tests

```yaml
# .github/workflows/test.yml
name: Test Q Code

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download kdb+
        run: |
          # The free on-demand license works for CI
          # Set QLIC to point to your license file
          wget -q https://files.kx.com/linuxx86.zip
          unzip -q linuxx86.zip -d ~/q
          echo "$HOME/q/l64" >> $GITHUB_PATH

      - name: Run Q tests
        env:
          QLIC: ${{ secrets.KX_LICENSE_ENCODED }}
        run: |
          echo "$QLIC" | base64 -d > ~/q/kc.lic
          q tests/test.q -q
```

## Version Control Practices

Q code in version control isn't different from any other code, but a few Q-specific practices help:

### `.gitignore` for Q Projects

```gitignore
# kdb+ generated files
*.log
*.idx
*.d

# Splayed table files (the data, not the schema)
sym
*.dat

# Q session state
.q
.z_history

# Build artifacts
*.so
*.dylib

# License files (never commit these)
*.lic
kc.lic
```

### Separating Schema from Data

Keep your table schemas in `.q` files under version control; keep the actual data files out:

```
project/
├── q/
│   ├── schema.q      # table definitions
│   ├── functions.q   # business logic
│   └── init.q        # startup script
├── tests/
│   └── test.q
└── data/             # not in git
    ├── hdb/
    └── rdb/
```

```q
/ schema.q — version controlled
trade: ([]
    time:  `timestamp$();
    sym:   `symbol$();
    price: `float$();
    size:  `long$()
    )

quote: ([]
    time:  `timestamp$();
    sym:   `symbol$();
    bid:   `float$();
    ask:   `float$();
    bsize: `long$();
    asize: `long$()
    )
```

## Dependency Management

Q has no package manager. The ecosystem is small enough that this isn't as painful as it sounds, but it's something to manage intentionally.

Common patterns:

```q
/ Explicit version tracking in a manifest
/ versions.q
\d .versions
kdbplus: "4.1.0"
kafkaq: "1.5.2"
websockets: "0.3.1"
\d .

/ Load dependencies with version checks
loadDep: {[name; minVersion]
    // Load the library
    @[system; "l ", string name; {[e] '"Failed to load ", string name, ": ", e}];
    // Check version if accessible
    $[minVersion <= .versions[name];
        -1 "Loaded ", string name, " v", string .versions[name];
        '"Version ", string name, " ", string .versions[name],
         " < required ", string minVersion]
    }
```

No lock files, no `requirements.txt`, no `Cargo.toml`. Manage versions explicitly and document them. It's not ideal. It's the reality.

## Profiling Q Code

For performance work, Q has a built-in profiler:

```q
/ Time a single expression
\t select from trade where date=.z.d, sym=`AAPL
/ Output: 12 (milliseconds)

/ Time over N iterations
\t:1000 count trade
/ Output: 3 (total ms for 1000 runs)

/ More detailed profiling with .Q.ts
.Q.ts select from trade where date=.z.d
/ Output: (time ms; space bytes)

/ Memory usage
.Q.w[]
/ `used`heap`peak`wmax`mmap`mphy`syms`symw
/ See current memory allocation
```

For deeper profiling — finding hotspots in complex Q code — the combination of `\t` on individual functions with binary search is often the fastest approach:

```q
/ Profile each step of a pipeline
stages: (
    "load";       {select from trade where date=.z.d};
    "filter";     {select from trade where date=.z.d, sym=`AAPL};
    "aggregate";  {select vwap:size wavg price by sym from trade where date=.z.d};
    "join";       {aj[`sym`time; ...; ...]}
    );

/ Time each stage
{-1 (x 0), ": ", string \t (x 1)[]} each stages
```

There's no flame graph generator for Q. Yet.
