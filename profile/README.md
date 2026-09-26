# Diavasis

**Making existing data consumable.**

Diavasis is an open-source organization built around a simple idea:

> Existing databases should support durable consumer groups without requiring applications to become streaming applications.

Useful data already lives in databases, often written by applications that existed long before anyone needed to consume it as a stream. Processing that data reliably should not always require new producer APIs, replication pipelines, or another system to hold a copy.

**The database is already there. Diavasis starts from there.**

## Diavasi

[**Diavasi**](https://github.com/diavasis/diavasi) is the core project: a durable consumer-group server for existing databases.

It turns an ordered query into a resumable, parallel-consumable stream without changing the application that writes the data.

```text
                  Existing applications
                           |
                         writes
                           v
                       Database
                           |
                     ordered query
                           v
                        Diavasi
                     Consumer group
                           |
                 +---------+---------+
                 |         |         |
                 v         v         v
             Consumer 1 Consumer 2 Consumer 3
```

The group owns the traversal. Consumers share the work. Diavasi distributes records, tracks acknowledgements, checkpoints progress, and reconstructs the traversal after failure. Each consumer does not need to scan the dataset independently.

The producer application does not need to know Diavasi exists.



## Server and it's client libraries

| Project | Role |
| --- | --- |
| [diavasi](https://github.com/diavasis/diavasi) | Core server and consumer-group implementation |
| [diavasi-c](https://github.com/diavasis/diavasi-c) | C client |
| [diavasi-dotnet](https://github.com/diavasis/diavasi-dotnet) | C# client |
| [diavasi-elixir](https://github.com/diavasis/diavasi-elixir) | Elixir & Beam client |
| [diavasi-go](https://github.com/diavasis/diavasi-go) |Golang client |
| [diavasi-java](https://github.com/diavasis/diavasi-java) | Java and JVM client |
| [diavasi-python](https://github.com/diavasis/diavasi-python) | Python client |
| [diavasi-client](https://github.com/diavasis/diavasi-client) | Native Rust client |
| [diavasi-zig](https://github.com/diavasis/diavasi-zig) | Zig client |



Client libraries are deliberately small: idiomatic interfaces for joining groups, consuming batches, and acknowledging work. The server owns traversal, dispatch, checkpointing, and recovery; clients should not become separate implementations of that logic.

Future possibilities include `diavasi-go`, `diavasi-js`, and `diavasi-dotnet`. These are potential additions, not a promise of availability.

Whatever language an application uses, consuming a Diavasi group should feel native to that language.


## The abstraction

```text
query + ordering contract + consumer group
                    |
                    v
durable, resumable, parallel-consumable stream
```

Start with a database connection and a query over an ordinary table or collection. Define a stable ordering and create a consumer group. Multiple workers can then process the results while sharing durable progress.

Records do not need to have been published as events. There is no requirement to migrate them into a new system first or add a Diavasi producer SDK.

Consider a table containing millions of records that several workers need to process. The database already knows how to execute the query. What is missing is coordination: distributing work, recording completion, and recovering when workers or the server stop.

**Diavasi supplies the consumer-group abstraction over data you already have.**

## Failure is part of the design

> Downtime is often less dangerous than silent data loss.

Diavasi is designed around recovery. When completion is uncertain, the preferred behavior is to pause, retry, and replay rather than silently skip work.

The default delivery model is **at least once**. A record may be delivered again after a failure, so consumers should tolerate duplicate processing, for example through idempotent operations. Acknowledgements and durable checkpoints establish the progress from which a group can safely resume.

Durable progress does not create an independent copy of the source data. Recovery depends on the source remaining available and satisfying its ordering contract.

## Logical cursors

A database cursor is useful while its connection is alive. It is not, by itself, a durable recovery mechanism.

Diavasi persists a **logical position in an ordered dataset**, such as:

```text
(id)
(created_at, id)
(sequence, partition, id)
```

After a restart, the database adapter reconstructs the query from the last durably committed position. The checkpoint represents recoverable group progress, not merely the furthest record read or dispatched.

This separates the lifetime of a consumer group from the lifetime of a database connection.

## The ordering contract

Reliable continuation requires a deterministic ordering, with a stable tie-breaker where values are not unique. The creator of a consumer group defines that ordering and the assumptions under which forward traversal is valid.

For example:

```sql
SELECT *
FROM records
ORDER BY created_at, id;
```

The contract matters:

- A record inserted behind an already committed position may not be discovered by normal forward traversal.
- Changing an ordering field can move a record across the checkpoint.
- Deleting source records can make them unavailable for replay.
- Ordered traversal does not imply that parallel consumers finish work in order.

Diavasi aims to validate what it can and make these assumptions explicit. It cannot impose immutable stream semantics on arbitrary mutable data. Reconciliation and anti-entropy mechanisms for records appearing behind a checkpoint are future work.

## Database adapters

Diavasi is designed around a database-neutral consumer-group core and database-specific traversal adapters. Initial adapter targets include **PostgreSQL, MongoDB, Redis, and ScyllaDB**.

These databases have different models for indexes, ordering, partitions, consistency, and efficient continuation. Adapters expose those constraints while translating native query and traversal capabilities into a common model for reading batches and resuming from logical positions.

The core owns dispatch, acknowledgements, checkpoint coordination, and recovery. See the [core repository](https://github.com/diavasis/diavasi) for implementation status and supported capabilities.

## Control plane and data plane

Administration and record delivery have different requirements.

The **control plane** manages connections, consumer groups, lifecycle, configuration, health, and observability. A conventional HTTP API keeps the CLI and operational tooling straightforward.

The **data plane** handles record batches, acknowledgements, flow control, reconnects, and backpressure. Its transport can be optimized independently, with choices guided by benchmarks.

Clients expose these capabilities in the conventions of their language. Consumer-group semantics remain the server's responsibility.

## Keep the server boring

Diavasi begins with one server instance and one isolated logical process per consumer group. Each group owns its query, buffering, dispatch, in-flight work, checkpoint, and recovery lifecycle.

```text
Diavasi server
|
+-- Control plane
|
+-- Consumer group A
|   +-- Reader and buffer
|   +-- Dispatcher and consumers
|   +-- Acknowledgements and checkpoint
|   +-- Recovery lifecycle
|
+-- Consumer group B
    +-- Independent state and lifecycle
```

The group is the unit of ownership, scheduling, and recovery. A logical process need not be a separate operating-system process.

Our priorities are explicit ownership, bounded memory, small durable state, simple state machines, deterministic recovery, measurable performance, and aggressive failure testing.

The initial architecture avoids distributed consensus and unnecessary clustering. More moving parts should follow demonstrated operational needs.

## What Diavasi is not

Diavasi is not a replacement for Kafka, NATS, Redis Streams, CDC platforms, or databases. It is not a transaction log of every source mutation, and it does not promise exactly-once application side effects.

Its focus is the space between:

> “I already have the data” and “I need multiple consumers to process it reliably.”

Sometimes CDC and a streaming platform are the right answer. Sometimes it is simpler to consume the data where it already lives. Diavasis exists to make that second option practical.


## Get involved

Start with the [Diavasi repository](https://github.com/diavasis/diavasi) or a client in your language. Bug reports, reproducible failure cases, adapter work, benchmarks, documentation, and API feedback all help build dependable infrastructure.

## The name

**Diavasis** comes from the Greek **διάβασις**, an older form associated with passage, crossing, or traversal. The core project, **Diavasi**, takes its name from the modern Greek **διάβαση**.

That is what the software is designed to provide: **a controlled, resumable passage through existing data.**
