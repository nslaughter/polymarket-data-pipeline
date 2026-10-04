# Polymarket data pipeline

A demonstration pipeline for retaining market-data events durably, batching
them into object storage, and recovering interrupted delivery.

I'm [Nathan Slaughter](https://nathanslaughter.com/), a software engineer with
experience building and operating data pipelines, observability systems, and
infrastructure. I help teams turn incoming data into stored datasets they can
inspect, query, and reprocess. This project will make the storage and recovery
decisions behind that work explicit.

**Status:** Project brief. This repository currently contains this README.
The pipeline, deployment configuration, storage format, and recovery examples
are planned; nothing described here has been implemented or tested yet. The
pipeline will be written in Python, like the client it builds on. This is an
independent demonstration using Polymarket market data. It is not affiliated
with or endorsed by Polymarket, and it is not client work.

## What this project demonstrates

This is the supporting example for my data ingestion and storage work: a
pipeline that collects API data into storage a team owns, with tested recovery
and a documented handover. It is meant to show:

- **A defined durability boundary.** Records are durably accepted at a stated
  point, before they wait for a storage batch.
- **Batches that survive retries.** Batch identity and membership persist
  across restarts, stored objects are immutable, and readers see only
  completed batches.
- **Recovery from interrupted delivery.** A process failure, a destination
  outage, an upload whose success response is lost, and a crash between the
  object write and its completion record each recover without losing or
  duplicating accepted records.
- **Operational signals.** Backlog age and remaining capacity are visible
  before the buffer's limits are reached.
- **Source uncertainty kept with the data.** Known capture interruptions and
  unresolved completeness travel with the stored dataset.
- **A pinned dependency.** The pipeline installs a tagged release of the
  Python client as a pinned package and checks its behavior when that release
  is upgraded.

The pipeline is the second of two stages, built on
[polymarket-market-data-client](https://github.com/nslaughter/polymarket-market-data-client).

## Waiting for a larger file should not leave accepted records only in memory

Object storage works naturally with completed files, while a WebSocket produces
events over time. The proposed pipeline places a durable event log between
collection and file delivery. Records can then wait for an efficient batch
without depending on the writer process staying alive.

The integration itself stays in the client repository. This pipeline's first
scope is one storage destination with explicit batch completion, retention
limits, and recovery behavior. The client's record identities are preserved
through to the stored files.

## How accepted events will become recoverable storage batches

1. Append each received record to the durable log, assigning its ingestion
   identity once and preserving it through replay and delivery retries.
2. Select a bounded range of accepted records when a byte-size threshold or
   maximum age is reached. Persist the batch identity and membership so a
   restart reconstructs the same batch.
3. Encode the records in the agreed file format and write an immutable object
   whose name stays stable on retry, so an uncertain upload resolves without a
   second logical batch.
4. Confirm the complete object and durably record its location, record range,
   count, checksum, and format version. Readers discover only batches with a
   completion record.
5. Advance the delivery checkpoint only after completion is durable, and
   release buffered records according to that checkpoint and the retention
   policy. After a crash, reconcile unfinished batches before continuing.

An object write and a separate completion record are not one transaction. The
recovery demonstration will interrupt the pipeline between them and show how an
existing object without a completion record is reconciled.

## What each stored record keeps

Each record keeps its original payload, source time where available, receipt
time, capture session, and stable ingestion identity. Malformed messages are
retained with an explicit processing outcome where feasible.

Duplicates arriving from the source stay in the raw capture unless a reliable
source identity supports a documented interpretation. Identical payloads can
represent different observations, so a payload hash alone is not grounds for
discarding one. Ingestion identities define this pipeline's replay behavior;
they make no claim about unique upstream events.

## Implementation choices will follow the failure model

- **The durable store.** A local synchronized journal covers process recovery.
  Surviving host or disk loss requires an independent durable copy with a
  suitable acknowledgement policy. Host-loss testing will be added only if that
  failure is in scope.
- **Managed delivery or a custom writer.** A managed service such as Amazon
  Data Firehose already groups records into S3 objects, with its own retention
  and duplicate behavior during failures. I'll compare it with a custom batch
  writer before selecting the implementation, and document the trade-off.
- **Batch size and age.** Limits are chosen from measured file size, maximum
  record waiting time, storage writes, and a representative query under the
  demonstration workload.

## Failure cases and the evidence each must produce

| Failure | Evidence to collect |
| --- | --- |
| Process stops before a batch is uploaded | Every durably accepted fixture record remains recoverable. |
| Upload succeeds but its response is lost | Recovery exposes one logical batch with the expected records. |
| Process stops between object write and completion record | The existing object is reconciled and the checkpoint advances safely. |
| Storage is unavailable within the agreed buffer budget | Backlog is visible and drains after recovery without missing accepted records. |
| Capacity or retention limit approaches | The operator receives the agreed signal and the resulting action is recorded. |
| Source connection is interrupted | Capture uncertainty remains visible even after storage delivery catches up. |

Fixture records prepared independently of the pipeline will be compared with
the recovered dataset, and a query over the stored files will check record
identities and values.

## Stored-data recovery and source coverage need separate evidence

Durability begins at a defined acceptance point and covers a defined set of
failures. Capacity and retention bound how long destination delivery can stop
while collection continues. Backlog age and remaining capacity therefore need
to be observable.

Events missed before acceptance depend on the source's recovery capabilities.
Replaying records already in the durable log is different from recovering
events the collector never accepted, and restoring current market state does
not recreate the intervening history. Known connection interruptions and
unresolved capture intervals will travel with the stored dataset. The
demonstration makes no claim of complete source capture or universal
exactly-once delivery.

## Scope

In scope: records from a tagged client release, one object-storage
destination, a readable file format, defined retention and capacity limits, and
the failures listed above.

Outside this demonstration: the integration itself (in the client repository),
multiple destinations, and continuous production operation.

## The demonstration is complete when

- Process failure, destination outage, ambiguous upload success, and
  interruption between object write and completion recording have each been
  exercised.
- The durably accepted fixture records reconcile after recovery, and the stored
  output can be queried.
- Backlog and capacity signals are demonstrated.
- Host loss is tested, if that failure is covered.
- The pinned client version is documented, and upgrading it preserves the
  pipeline's expected behavior.

## What the repository will contain

- This README, explaining the retained dataset and its use.
- A reproducible startup command and deployment configuration.
- A data dictionary, and documentation of batches and checkpoints.
- Recovery commands, automated fault checks, and inspectable results.
- An example query or replay over the stored files.
- A tagged release, with documented capacity, retention, and source-coverage
  limits and the client release it uses.

## Related projects and writing

- [polymarket-market-data-client](https://github.com/nslaughter/polymarket-market-data-client):
  the read-only integration this pipeline consumes.
- *How to batch WebSocket data into object storage and recover interrupted
  writes*: an article on this pipeline's design, in preparation. I'll link it
  here when it is published.

## Work with me on a dataset your team can retain and use

I build ingestion and storage pipelines around an agreed source, destination,
and research or analysis workflow. Projects can include the collector, durable
buffering, batched delivery, deployment, operational checks, and maintenance.

[Discuss a data ingestion project](https://www.linkedin.com/in/nathan-slaughter)
with the source, intended storage, approximate volume, and freshness and
recovery needs.
