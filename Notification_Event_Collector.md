# Notification Event Collector

## Overview

The **Notification Event Collector** is a reusable event-processing pattern within the **ST Integration Framework**, developed by ST Consultancy B.V. for SAP Cloud Integration.

Its purpose is to collect notifications about changes to business objects and consolidate them before triggering downstream processing. This is particularly useful when source systems publish many events for the same object in a short period, potentially causing redundant API calls and unnecessary load on SAP backends.

> **Implementation note:** This document describes the intended pattern and recommended design. Queue names, timing intervals, event payload structures, and collector behavior must be confirmed against the deployed `ST_Collector_Events` artifact before using it as an implementation specification.

## Objectives

- Decouple event reception from backend processing.
- Reduce duplicate or closely spaced notifications for the same business object.
- Preserve enough event context to identify the affected object and route the request.
- Improve resilience during bursts of events.
- Avoid unnecessary backend API calls when multiple notifications can be consolidated.

## High-Level Architecture

```text
Event Producer / Notification Source
              |
              v
     Event Ingestion iFlow
              |
              v
    Event Collection / Buffer
              |
              v
     Consolidation by Key
              |
              v
       Downstream Router
              |
              v
    Receiver iFlow / API Call
              |
              v
       Target SAP System
```

The collection/buffer and consolidation stages are logical components. Their concrete implementation (for example, JMS, Data Store, or another persistence mechanism) depends on the deployed collector design.

## Processing Concept

### 1. Receive notifications

An integration flow receives an event or notification from an event source, such as SAP Event Mesh or another event publisher. The message should identify the business object and the operation or change that occurred.

### 2. Identify the consolidation key

Derive a stable key that represents the logical object to process. A key may combine:

- Source system
- Event or business-object type
- Business-object identifier

For example:

```text
SAPS4H|Equipment|11565325
```

The key must be chosen carefully: events for different objects or tenants must never be combined accidentally.

### 3. Collect and consolidate

Hold or group notifications according to a defined policy, such as a configurable time window or until a processing threshold is reached. Where appropriate, multiple notifications for the same key may be represented by one downstream processing request.

**Consolidation is not appropriate for every event.** It is suitable when notifications signal that the latest object state should be retrieved. It may be unsuitable for transactional events where each occurrence carries independent business meaning and must be processed separately.

### 4. Trigger downstream processing

Once an object is ready for processing, send a request to the appropriate router or receiver iFlow. In a state-retrieval pattern, the receiver calls the source API to fetch the current object details rather than relying solely on the notification payload.

### 5. Handle failures

Failures must follow the framework's receiver error-handling approach. Preserve the object key, source reference, and correlation information so failed requests can be diagnosed and recovered.

## Example: Equipment Change Notifications

A source system may publish several change notifications for the same equipment within a short interval:

```text
Equipment 11565325 changed (event 1)
Equipment 11565325 changed (event 2)
Equipment 11565325 changed (event 3)
                 |
                 v
        Notification Collector
                 |
                 v
      Equipment 11565325 (once)
                 |
                 v
        Retrieve latest state
```

This can reduce repeated calls when the downstream requirement is to synchronize the **latest state** of the equipment. It must not be used to discard meaningful intermediate state transitions when every transition is required.

## Design Considerations

### Consolidation semantics

Define whether the collector performs deduplication, debouncing, coalescing, or batching. These behaviors are different:

| Behavior | Meaning |
|---|---|
| Deduplication | Suppress repeated delivery of the same logical event |
| Debouncing | Wait for a quiet period before processing an object |
| Coalescing | Combine multiple object-change notifications into one processing request |
| Batching | Process multiple distinct objects together |

### Delivery and reliability

- Do not acknowledge a notification before the selected design has durably accepted it, unless loss is explicitly acceptable.
- Document retention periods, retry behavior, and what happens after a runtime restart.
- Avoid losing the final notification when updates arrive continuously.
- Consider concurrent collector instances and race conditions for the same key.
- Define how delayed or out-of-order notifications are handled.

### Throughput and backend protection

Consolidation reduces unnecessary work but does not itself guarantee a safe backend request rate. Where necessary, combine it with rate limiting or surge protection in the downstream processing path.

### Monitoring

Recommended diagnostic information includes:

- Source system and event type
- Business-object identifier / consolidation key
- Correlation or event identifier
- Time of first and latest collected notification
- Number of notifications consolidated, if tracked
- Downstream processing outcome

Avoid recording sensitive payload data unless it is needed and permitted.

## Integration with Other Framework Components

The collector can be combined with:

- **Global Router iFlows:** standardized dispatching to receiver integrations.
- **Receiver Error Handling:** classification, retry control, and failed-message recovery.
- **JMS-based asynchronous processing:** decoupling producers from consumers where configured.
- **ST Integration Toolkit (commercial):** optional operational applications for error handling and audit logging.

## Configuration Checklist

Before deploying the collector, confirm:

1. Event source and subscription configuration.
2. Event schema and extraction of business-object identifiers.
3. Consolidation key and separation of tenants/source systems.
4. Collection interval or trigger conditions.
5. Persistence and restart behavior.
6. Downstream routing destination.
7. Retry, failure, and recovery procedures.
8. Monitoring fields and retention requirements.
9. Whether business requirements allow intermediate notifications to be consolidated.

## Related Documentation

- [ST Naming Convention](ST_Naming_Convention.md)
- [Receiver Error Handling](Receiver_Error_Handling.md)
- [Sender Exception Handling](Sender_Exception_Handling.md)

## License

Copyright 2026 ST Consultancy B.V.

Licensed under the Apache License, Version 2.0. See the repository's [LICENSE](LICENSE) file for details.

This is an independent project and is not an official SAP product.
