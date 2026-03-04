# Where to Go From Here

You now have a map of the integration landscape. Let's be direct about what you actually need.

## The Honest Summary

**If you're building a data pipeline that feeds kdb+ from Kafka**: Use the Python Kafka bridge pattern from the Streaming chapter. It's the lowest-friction path, and Python's Kafka client ecosystem is mature. If throughput demands it, graduate to the native KafkaQ library.

**If your Python team needs kdb+ data**: Give them the FastAPI gateway from the REST chapter. They get a standard REST API with OpenAPI docs, you retain control over what they can query. `pykx` if they need direct access; the REST API if they just need data.

**If you're doing R-based research on kdb+ data**: `rkdb` plus the "aggregate in Q, analyze in R" pattern. The failure mode is pulling raw tick data into R. Don't do that.

**If you're building high-performance infrastructure around kdb+**: Rust via `kdbplus`. IPC for most use cases; FFI only when you need to write kdb+ extensions.

**If your IDE situation is still "text editor and a REPL"**: VS Code with the KX extension. Configure it with a connection to your dev kdb+ process. The interactive execution and diagnostics are worth the fifteen minutes of setup.

## What We Didn't Cover

A few areas deliberately left out:

**C integration**: The kdb+ C API is comprehensive and well-documented by KX. If you're writing a feed handler in C, the KX documentation is more authoritative than anything here.

**Java integration**: `jdbc` for kdb+ exists and works. If your organization is a Java shop and you need Q integration, the JDBC driver is maintained by KX and integrates cleanly with standard Java database frameworks.

**KX Insights and KDB.AI**: KX's commercial products have their own integration story. This book covers the open-source and community tooling; the commercial products have their own documentation.

**kdb+ on cloud infrastructure**: Deploying kdb+ on AWS/GCP/Azure has become more tractable with containers and KX's cloud products. The integration patterns here work the same in cloud environments; the operational considerations are different.

**Time-series ML with kdb+**: The combination of kdb+ tick data and Python ML libraries (scikit-learn, PyTorch, etc.) is a whole book. The PyQ and pykx chapters give you the foundation.

## The Patterns That Matter

Stepping back, three architectural patterns appear throughout this book:

**Pattern 1: Q as the source of truth, everything else as a consumer**

```
Tick → kdb+ → [Python analytics] [R research] [REST API] [WebSocket dashboard]
```

Q does the heavy lifting: ingestion, storage, time-series queries. External tools read from it. The integration layer (FastAPI, qPython, rkdb) is thin and stateless.

**Pattern 2: Q as one node in a broader pipeline**

```
Kafka → kdb+ → Kafka → [downstream services]
```

Q participates in a message-passing architecture. It consumes from Kafka, enriches or aggregates, and publishes back. Good for organizations that have already standardized on event streaming.

**Pattern 3: Q embedded in a polyglot service**

```
Rust service → kdb+ (IPC) → Rust service
             → kdb+ (FFI) ←
```

A Rust (or Go, or C++) service uses kdb+ as a compute or storage engine, calling it via IPC or embedding it via FFI. The kdb+ component is not independently managed; it's part of the service.

Most real systems are a hybrid. Know which pattern you're using for each integration point.

## The Community

Q's community is small but substantive:

- **KX Community** ([community.kx.com](https://community.kx.com)): Official forum. KX engineers answer questions. Good signal-to-noise ratio.
- **kdb+/q Insights** ([kdb.ai](https://kdb.ai)): KX's developer portal, tutorials, and documentation.
- **GitHub**: Search `kdb` or `kdbplus` — there's more community tooling than the sparse ecosystem reputation suggests.
- **Stack Overflow**: `[kdb]` and `[q-lang]` tags. Smaller than Python/JS but answers exist for most common questions.

## A Note on KX Licensing

The free 64-bit on-demand license from KX covers everything in this book. The license requires internet connectivity for validation. For air-gapped or cloud deployments, you need a commercial license.

The business reality: if kdb+ is generating value in your organization, you likely already have a commercial relationship with KX. If you're evaluating or building with the free license, be aware of the validation requirement before you plan a production deployment.

## One More Thing

Q developers who complain that the language is too opaque, the tooling is immature, and the community is too small are usually right on all three counts. They are often also the people who have been quietly building the most performant time-series systems in their organizations for a decade.

The gap between Q's ergonomics and Q's performance is real and it's shrinking. The IDE support is better than it was. The integration libraries are better than they were. The free license tier exists now. None of that was true five years ago.

What makes Q worth the friction is what has always made it worth the friction: when you need to query a billion-row time-series table and get a result in milliseconds, there are very few alternatives that aren't either considerably more expensive, considerably more complex to operate, or both. That's a narrow but deep niche, and it's not going away.

The rest of the stack — Python, Rust, R, REST, Kafka, WebSockets — is better at what it's good at. The goal of this book was to make Q a first-class participant in that stack rather than a silo. Whether you got there depends on what you built.

Go build something.
