# Distributed File Compression System: Case Study

> A multi-threaded client-server compression service in modern C++.
> TCP sockets, a task pool built on `std::future`, two interchangeable compression algorithms
> behind a polymorphic interface, and 14 test executables.

**Module:** CMP9133 Programming Principles, MSc Computer Science, University of Lincoln
**Assessment:** Assessment 2, Distributed File Compression System

> **Why this repository has no source code.** The implementation lives in a GitHub Classroom
> repository owned by the university, and the assessment brief is reused with future cohorts.
> This is a public case study covering architecture, design decisions, and test strategy.
> The source is available privately on request.

---

## The problem

Build a service where clients send files to a server for compression or decompression over TCP.
The brief required object-oriented design (inheritance, polymorphism, encapsulation), genuine
concurrency, a client-server model over sockets, and a codebase that stays modular and extensible.

The baseline requirement was one algorithm, Run-Length Encoding. I implemented two.

---

## Architecture

```
        ClientMain                                ServerMain
            │                                          │
    ┌───────▼────────┐                        ┌────────▼────────┐
    │ Client         │   TCP socket           │ Server          │
    │  read file     │ ─────────────────────► │  accept loop    │
    │  send op+data  │                        │  (single thread)│
    │  recv result   │ ◄───────────────────── │        │        │
    │  write file    │                        │        ▼        │
    └────────────────┘                        │  ThreadPool     │
                                              │  enqueue(task)  │
                                              │  to std::future │
                                              └────────┬────────┘
                                                       │
                                       ┌───────────────┼───────────────┐
                                       ▼               ▼               ▼
                                   worker 0        worker 1   …   worker N-1
                                       │               │               │
                                       └───────────────┼───────────────┘
                                                       ▼
                                         AlgorithmFactory::create(type)
                                                       │
                                    ┌──────────────────┴──────────────────┐
                                    ▼                                     ▼
                        RLECompression                         HuffmanCompression
                                    └──────────────────┬──────────────────┘
                                                       ▼
                                        CompressionAlgorithm (abstract)
                                     compress() / decompress(), pure virtual

              Cross-cutting: FileHandler · Config · Logger · ServerStatistics
```

### Layout

```
include/
  CompressionAlgorithm.h      abstract base, pure virtual compress/decompress
  RLECompression.h            run-length encoding
  HuffmanCompression.h        frequency tree, prefix codes
  AlgorithmFactory.h          runtime selection by type
  Client.h  Server.h          network endpoints
  FileHandler.h               binary-safe file I/O
  server/ThreadPool.h         variadic-template task queue over std::future
  server/ServerStatistics.h   thread-safe counters
  utils/Config.h              runtime configuration
  utils/Logger.h              synchronised logging
src/                          implementations plus ClientMain / ServerMain
tests/                        14 test executables
CMakeLists.txt                build
```

---

## Technical decisions

*Chose X over Y because Z.*

- **Chose a thread pool over one thread per connection.**
  Thread-per-connection is fewer lines and it is what the brief's baseline implies. It also means
  an unbounded number of OS threads, and the point at which that degrades is exactly the point the
  scalability test measures. A fixed pool sized to `std::thread::hardware_concurrency()` bounds
  resource use, amortises thread creation across requests, and turns "how many clients can it take"
  into a queueing question rather than a crash.

- **Chose `std::future` as the task-result channel over a shared results structure with a mutex.**
  `enqueue()` is a variadic template using perfect forwarding, returning
  `std::future<std::result_of_t<F(Args...)>>`. The caller gets a typed handle and the
  synchronisation becomes the standard library's problem rather than mine. It also means an
  exception thrown inside a worker propagates to `.get()` instead of being swallowed on a detached
  thread.

- **Chose to delete copy and move on `ThreadPool` rather than implement them.**
  The pool owns running threads, a mutex, and a condition variable. There is no correct move
  semantic for an object whose workers hold references to its own queue, and a defaulted move would
  compile and then be undefined behaviour at runtime. Deleting all four makes the misuse a compile
  error. The destructor joins.

- **Chose a factory returning the abstract base over branching on algorithm type at each call site.**
  `CompressionAlgorithm` declares `compress` and `decompress` as pure virtual over
  `std::vector<char>`. `AlgorithmFactory` maps a runtime type to a concrete instance. Adding a third
  algorithm touches the factory and nothing else, so the server, the client protocol, and every test
  that exercises polymorphism stay unchanged. This is what "extensible" meant in the brief, made
  concrete.

- **Chose `std::vector<char>` over `std::string` as the data currency throughout.**
  Compressed output is arbitrary binary and contains embedded nulls. `std::string` works until it
  silently does not, and the failure appears as truncated files rather than as an error. A byte
  vector makes the binary nature explicit at every interface boundary, including file I/O.

- **Chose a virtual destructor on the abstract base.**
  Instances are held and deleted through base pointers. Without it, deleting a `HuffmanCompression`
  through a `CompressionAlgorithm*` is undefined behaviour and leaks the derived members. One line,
  and the kind of line whose absence is invisible until it is not.

---

## Test strategy

14 test executables, grouped by what they are actually trying to break:

| Concern | Tests |
|---|---|
| Correctness, round trip | RLE round trip, RLE on empty input, `FileHandler` round trip |
| Correctness, Huffman | Comprehensive Huffman suite, algorithm comparison |
| Design | Polymorphism through the base pointer |
| Networking | Single client, multiple concurrent clients |
| Concurrency | Shared-counter race test, concurrent compression, thread-pool server |
| Scale and robustness | Large-file scalability, edge cases, additional coverage |

The two that mattered most were the shared-counter race test, which is designed to fail loudly if
`ServerStatistics` is not properly synchronised, and the multiple-client network test, which is the
only thing that actually proves the pool is doing what the design claims.

---

## What I would do differently

- The wire protocol is minimal, an operation byte plus payload. It has no length-prefixed framing
  for multi-message sessions and no version field, so extending it would be a breaking change.
- Huffman rebuilds its frequency table per request. For repeated compression of similar files, a
  cached or canonical table would cut work substantially.
- `Config` is read at startup only, so there is no reload without a restart.

---

## Author

**Temitope Alabi**, MSc Computer Science (AI), University of Lincoln
[GitHub](https://github.com/toptech5419) · [LinkedIn](https://www.linkedin.com/in/toptech5419/) · alabitemitope51@gmail.com
