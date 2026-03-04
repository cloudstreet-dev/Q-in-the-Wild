# Q in the Wild

> A kdb+/q developer's guide to integrating with the rest of the world.

You know Q. You know it's fast. You know it eats time-series data for breakfast and asks for seconds. What you might not know is how to make it talk to Python without wanting to flip a table, how to expose it as a REST API without crying, or why your Rust FFI attempt keeps segfaulting in a way that feels personal.

This book fixes that.

**Q in the Wild** is a practical, code-first guide to integrating kdb+/q with everything else: Python, Rust, R, web frameworks, streaming platforms, REST APIs, and the IDE that won't make you feel like it's 2003. It assumes you know Q. It doesn't assume you know everything else.

---

## Table of Contents

1. [Q in the Wild: Breaking Out of the Silo](src/introduction.md)
2. [Python and Q: PyQ, qPython, and Getting Along](src/python.md)
3. [Rust and Q: FFI, IPC, and High-Performance Integration](src/rust.md)
4. [R and Q: Statistical Power Meets Time-Series Performance](src/r.md)
5. [IDE Support: What Actually Works in 2026](src/ides.md)
6. [Web Frameworks and Q: Serving Data to the Outside World](src/webframeworks.md)
7. [Q IPC Deep Dive: The Protocol, the Patterns, and the Pitfalls](src/ipc.md)
8. [REST and Q: Building APIs Around kdb+/q](src/rest.md)
9. [Streaming Integrations: Kafka, WebSockets, and Real-Time Q](src/streaming.md)
10. [Developer Tooling: Linters, Formatters, and Making Q Less Lonely](src/devtools.md)
11. [Where to Go From Here](src/conclusion.md)

---

## Read Online

The book is published at: **[https://cloudstreet-dev.github.io/Q-in-the-Wild/](https://cloudstreet-dev.github.io/Q-in-the-Wild/)**

---

## Building Locally

### Prerequisites

Install [mdBook](https://rust-lang.github.io/mdBook/guide/installation.html):

```bash
cargo install mdbook
```

Or via a package manager:

```bash
# macOS
brew install mdbook

# Linux (via cargo)
cargo install mdbook
```

### Build

```bash
git clone https://github.com/cloudstreet-dev/Q-in-the-Wild.git
cd Q-in-the-Wild
mdbook build
```

The output is in `./book/`. Open `book/index.html` in a browser.

### Serve with Live Reload

```bash
mdbook serve --open
```

This starts a local server at `http://localhost:3000` and rebuilds on file changes. Useful when writing or editing chapters.

### Clean

```bash
mdbook clean
```

---

## Contributing

Found a bug? An outdated library version? Code that doesn't compile because a crate changed its API again? Open an issue or PR.

The bar for contributions: code must actually run, explanations must be accurate, and humor must be earned.

---

## License

MIT. Use it, build on it, ship it.

---

*A CloudStreet production. Technical, accurate, and genuinely amusing.*
