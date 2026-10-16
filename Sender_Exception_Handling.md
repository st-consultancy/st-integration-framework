# Sender Exception Handling

**ST Integration Framework**  
Developed and maintained by **ST Consultancy B.V.**

## Purpose
Sender exception handling provides a consistent way to diagnose and recover from failures occurring in a sender integration flow, particularly when a message cannot be handed over reliably to the next processing stage (such as a JMS queue or ProcessDirect receiver).

The implementation depends on the sender protocol. **HTTP** requests can receive an explicit error response and may require preservation of the original request. **IDoc** messages can generally be investigated and recovered from the sending SAP system, so the error should be propagated rather than acknowledged as successful.

## Scope
This guide covers:
- Standardized sender exception monitoring in the Message Processing Log (MPL).
- HTTP sender exception handling with a diagnostic Data Store entry and structured HTTP response.
- IDoc sender exception handling with failure propagation and backend recovery.
- Recommended monitoring fields and operational recovery procedures.

Receiver-side business and technical errors, automatic retries, and receiver Error Data Stores are documented separately.

## Common monitoring conventions
| Field | Convention / purpose |
| --- | --- |
| MPL custom status | `ST_SenderException` |
| Application Message ID | Business identifier when available (for example, external customer ID or IDoc number) |
| Application Message Type | Interface message type |
| Sender / Receiver | Logical source and destination systems |
| `st_errorMessage` | Short, readable exception summary |
| `st_errorDetails` | Recovery information or Data Store reference |
| Sender exception Data Store | `STexp_SenderException` by default |
| Data Store entry ID | MPL Message ID, allowing correlation from the response to monitoring and the stored diagnostics |

> **Important:** A sender iFlow may show technical MPL status **Completed** when its exception subprocess intentionally catches an error and returns an HTTP error response. Use the custom status `ST_SenderException` and the HTTP response code to identify the failed business request. An IDoc sender exception that ends with an **Error End Event** should instead propagate the failure.

## 1. HTTP sender exception handling

### Processing pattern

```text
HTTP Client
    |
    v
Sender iFlow -- handoff to next processing stage --> JMS / ProcessDirect
    |
    +-- Exception Subprocess
          |
          +-- Capture original request and exception diagnostics
          +-- Set MPL custom status and correlation headers
          +-- Persist diagnostics to STexp_SenderException
          +-- Return structured HTTP error response
```

### Recommended exception subprocess
1. **Capture diagnostics.** Prepare a diagnostic body containing the original request payload, any available receiver response payload, the exception message, and stack trace. Avoid recording secrets or sensitive payloads without appropriate controls.
2. **Set monitoring metadata.** Set MPL custom status to `ST_SenderException`, populate `st_errorMessage`, and record the Data Store name and entry ID in `st_errorDetails`.
3. **Persist diagnostics.** Write the diagnostic body to `STexp_SenderException` using the MPL Message ID as the entry ID.
4. **Return an error to the caller.** For an unavailable asynchronous handoff, return **HTTP 503 Service Unavailable** with a structured error payload and the MPL Message ID. For a synchronous flow, return an appropriate status reflecting the failure; a receiver validation error may be **HTTP 400 Bad Request**.
5. **Handle failures in the exception handler.** If diagnostic persistence itself fails, use a fallback error response rather than falsely reporting success. The fallback may not have a stored diagnostic entry.

### Example HTTP error response
```xml
<root>
  <status>503</status>
  <message_id>AGrR8IaGIze8S536UVPRmkVVaJfs</message_id>
  <messages>
    <item>
      <type>E</type>
      <code>SE001</code>
      <message>Sender processing failed. Refer to the Message Processing Log.</message>
    </item>
  </messages>
</root>
```

`message_id` is the correlation key for locating the corresponding MPL entry and, if persistence succeeded, its Data Store diagnostic record. The example identifier is illustrative.

### Diagnostic Data Store content
The stored body can be organized into four sections:

```text
----- Original Request Payload -----
<original inbound payload>

----- Receiver Response Payload -----
<receiver response, if available>

----- Exception Message -----
<exception summary>

----- Exception Stack Trace -----
<exception stack trace>
```

For example, when an OData V2 receiver rejects an invalid country code, the receiver response section might contain:
```xml
<error xmlns="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata">
  <code>SEPM_BO_COMMON/046</code>
  <message xml:lang="en">Invalid country code 'DX'</message>
  <innererror>...</innererror>
</error>
```

### Operational recovery (HTTP)
1. Obtain `message_id` from the caller's error response.
2. Search the SAP Cloud Integration Message Processing Logs using that ID.
3. Confirm the custom status `ST_SenderException` and inspect `st_errorMessage` and `st_errorDetails`.
4. Open the indicated Data Store and entry ID (normally `STexp_SenderException` and the MPL Message ID).
5. Review the original request, receiver response, and exception details.
6. Correct the cause and resubmit the original request through the appropriate supported mechanism. Verify that resubmission will not create duplicates.

> A diagnostic Data Store entry is **not** automatically a retry queue. Recovery and resubmission require a defined operational procedure or tooling.

## 2. IDoc sender exception handling

### Processing pattern
```text
SAP Backend -- IDoc --> Sender iFlow --> JMS Queue
                            |
                            +-- Exception Subprocess
                                  +-- Set ST_SenderException
                                  +-- Record IDoc number and recovery guidance
                                  +-- Error End Event
                                           |
                                           v
                              Failure propagated to sender adapter/backend
```

The objective is to avoid acknowledging an IDoc as successfully handed off when JMS persistence or another critical sender step fails.

### Recommended exception subprocess
1. Set MPL custom status to `ST_SenderException`.
2. Record the IDoc number as the Application Message ID and an MPL custom header when available.
3. Set the Application Message Type (for example, `FLIGHTCUSTOMER_CHANGE`) and source system (for example, `SAPS4H`).
4. Record the exception in `st_errorMessage` and include recovery guidance such as `Message Recovery = Check SAP backend using IDoc number.`
5. End with an **Error End Event** so that the adapter can propagate failure rather than treating the inbound IDoc as successfully processed.

**Do not automatically create a sender exception Data Store entry for this IDoc pattern.** The original IDoc is generally available in the sending SAP system, which should be the recovery source. Verify actual backend IDoc status and adapter behavior in the relevant landscape.

### Operational recovery (IDoc)
1. Find the failed MPL using the IDoc number, Application Message ID, or custom header.
2. Inspect the exception and verify whether the JMS handoff succeeded.
3. In the sending SAP system, locate the IDoc with transaction `WE02` or `WE05`.
4. Resolve the underlying issue (for example, JMS queue unavailable or full).
5. Check the IDoc status and reprocess using the appropriate backend procedure, such as `BD87` **when applicable**.
6. Confirm that exactly one message has reached the downstream processing stage before attempting further reprocessing.

> SAP backend status codes and reprocessing options depend on the IDoc direction, port, adapter, and failure point. Do not assume every failed sender handoff can be restarted with `BD87`.

## 3. Failure scenarios
| Scenario | Expected sender behavior | Recovery source |
| --- | --- | --- |
| HTTP sender cannot write to JMS | Return 503; mark `ST_SenderException`; preserve diagnostics if possible | HTTP request / sender exception Data Store |
| HTTP synchronous receiver returns validation error | Return appropriate 4xx response; preserve available diagnostics | Original request and MPL / Data Store |
| IDoc sender cannot write to JMS | Propagate failure using Error End Event; record IDoc number | Sending SAP system |
| Sender exception Data Store write fails | Return fallback HTTP error; do not claim diagnostics were saved | MPL and caller's original request |
| Receiver fails after successful JMS handoff | Handle in receiver/global router flow, **not** as sender exception | Receiver monitoring / Error Data Store |

## 4. Implementation and security considerations
- **Acknowledge only after durable handoff.** For asynchronous senders, do not report successful receipt before the JMS write or equivalent durable handoff has succeeded.
- **Keep monitoring searchable.** Use consistent Application Message IDs, message types, custom status, and error metadata.
- **Do not confuse transport success with business success.** A completed MPL may represent a handled HTTP error, not a successful business transaction.
- **Control diagnostic payload retention.** Configure Data Store retention and access permissions to match customer security requirements. Mask credentials, tokens, and personal or confidential data.
- **Plan for duplicates.** A timeout or lost acknowledgment can leave processing outcome uncertain. Apply appropriate idempotency and verify message state before replay.
- **Separate sender and receiver failures.** Once a message is durably placed on JMS, downstream errors belong to receiver/global-router error handling.
- **Test failure propagation.** Validate actual HTTP status codes, IDoc statuses, MPL fields, Data Store entries, and recovery steps in the deployed environment.

## 5. Related conventions and components
- [`ST_Naming_Convention.md`](ST_Naming_Convention.md) — artifact, Data Store, script, and header naming.
- `ST | Integration Accelerators` — reusable sender, router, and exception-handling patterns.
- `ST | Integration Toolkit` — optional commercial tools for operational error handling, audit logging, and integration testing.

## License
Part of the **ST Integration Framework** developed by ST Consultancy B.V.

Copyright 2026 ST Consultancy B.V.  
Licensed under the Apache License, Version 2.0. See the repository [`LICENSE`](LICENSE) file.

SAP and related product names are trademarks of SAP SE or its affiliates. This is an independent project and not an official SAP product.