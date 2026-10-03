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
are planned. This is an independent demonstration using Polymarket market data.

## Waiting for a larger file should not leave accepted records only in memory

Object storage works naturally with completed files, while a WebSocket produces
events over time. The proposed pipeline places a durable event log between
collection and file delivery. Records can then wait for an efficient batch
without depending on the writer process staying alive.

The pipeline will consume a versioned release of
[polymarket-market-data-client](https://github.com/nslaughter/polymarket-market-data-client).
Its first scope is one storage destination with explicit record identities,
batch completion, retention limits, and recovery behavior. The durable store
and delivery implementation will be selected against the failures and workload
the demonstration is intended to cover.

## How accepted events will become recoverable storage batches

1. Append received records to the durable log with stable ingestion identities,
   original payloads, and available source and receipt times.
2. Form a batch when its size threshold or maximum waiting time is reached.
   Preserve its identity and membership across retries.
3. Write the completed object, confirm its contents, and durably record the
   batch's completion before advancing the delivery checkpoint.
4. After an interruption, reconcile unfinished delivery and expose each
   completed batch once in the logical dataset.

The recovery demonstration will include an upload that succeeds while its
response is lost, a crash before completion is recorded, and a temporary
destination outage. Independently prepared fixture records will make it
possible to compare the recovered dataset with what was durably accepted.

## Stored-data recovery and source coverage need separate evidence

Durability begins at a defined acceptance point and covers a defined set of
failures. Capacity and retention bound how long destination delivery can stop
while collection continues. Backlog age and remaining capacity therefore need
to be observable.

Events missed before acceptance depend on the source's recovery capabilities.
Known connection interruptions and unresolved capture intervals will travel
with the stored dataset. The planned handover includes a data dictionary,
query or replay example, recovery commands, fault results, and those limits.

## Work with me on a dataset your team can retain and use

I build ingestion and storage pipelines around an agreed source, destination,
and research or analysis workflow. Projects can include the collector, durable
buffering, batched delivery, deployment, operational checks, and maintenance.

[Discuss a data ingestion project](https://nathanslaughter.com/) with
the source, intended storage, approximate volume, and freshness and recovery
needs.
