# Receiver Error Handling

**ST Integration Framework**  
Developed and maintained by **ST Consultancy B.V.**

## 1. Purpose

This guide describes a reusable approach to handling errors in **receiver integration flows** in SAP Cloud Integration. Receiver error handling distinguishes functional failures from technical failures, supports controlled automatic retries, and preserves diagnostic information for manual investigation and recovery.

Receiver-side failures are handled independently from **sender exceptions**. In an asynchronous JMS-based integration, an error in the receiver iFlow does not automatically mean that the original sender request failed.

## 2. Reference Architecture

```text
Source System
     |
     v
Sender iFlow
     |
     v
JMS Queue (e.g. Stage_SAPS4H)
     |
     v
GLOBAL_Router_SAPS4H
     |
     +---- ProcessDirect ----> IF0000_Receiver_SAPS4H
     |                               |
     |                               v
     |                          Target System
     |                               |
     |                         Success / Exception
     |                               |
     +<------------------------------+
     |
     +-- Retry eligible? --> Controlled retry (up to configured limit)
     |
     +-- Otherwise -------> Error Data Store (e.g. STerr_SAPS4H)
                                      |
                                      v
                               Manual investigation
                               and recovery
```

The diagram illustrates the logical pattern. Actual retry behavior and Data Store persistence depend on the deployed global router and its configuration.

## 3. Responsibilities

| Component | Responsibility |
|---|---|
| Sender iFlow | Receive and validate the incoming request; hand off asynchronous messages to JMS where applicable. |
| Global Router | Dispatch to the receiver iFlow, coordinate retry policy, and handle exhausted or non-retryable failures. |
| Receiver iFlow | Transform the message, invoke the target, validate the response, and communicate error classification to the router. |
| Error Data Store | Retain failed-message information for investigation and manual recovery. |
| ST Integration Toolkit (optional) | Provide an operational UI for inspecting, correcting, and resubmitting failed messages. |

## 4. Error Classification

### 4.1 Functional errors

Functional errors arise when the receiver rejects the business content, for example:

- Invalid country or currency code
- Missing mandatory field
- Invalid business state or unsupported operation
- Business validation error returned in an otherwise successful HTTP response

**Default treatment:** do not automatically retry unless a specific backend error code has been explicitly classified as transient. Persist the failure for correction and manual recovery.

### 4.2 Technical errors

Technical errors may include:

- Temporary backend unavailability
- Connection reset or transient network failure
- Receiver rate limiting or overload
- HTTP 5xx response
- Lock or concurrency conflict

**Default treatment:** retry only when the error is classified as transient and the operation can be retried safely. A status code alone does not establish retry safety.

### 4.3 Timeouts and uncertain outcomes

A timeout is different from a confirmed failure. The backend may have completed the operation even though Cloud Integration did not receive a response.

- Classify a receiver/gateway timeout internally as **504 Gateway Timeout** when appropriate.
- Preserve the original request identity across attempts.
- Enable automatic retry only if the receiver operation is idempotent or the backend provides verified duplicate protection.
- Otherwise route the message to investigation/reconciliation rather than blindly resubmitting it.

For asynchronous interfaces, the internal error classification is **not** an HTTP 504 response to the original sender.

## 5. Framework Configuration

The following are framework conventions. Verify whether a given value is a **header** or an **exchange property** in the deployed artifact.

| Name | Typical use |
|---|---|
| `st_maxRetry` | Maximum permitted automatic retries; configured as a header in newer receiver examples. |
| `st_autoRetry` | Indicates whether the classified error is eligible for automatic retry. |
| `st_autoRetryErrorCodes` | Pipe-separated list of retryable application or technical error codes. |
| `st_functionalErrorCodes` | Optional list used to identify functional errors, if supported by the deployed validator. |
| `st_errorMessage` | Concise error description for monitoring and troubleshooting. |
| `st_errorDetails` | Additional diagnostics, such as backend error details or Data Store references. |
| `st_errorResponseCode` | Internal HTTP-style error classification where the framework uses one. |
| `st_logPayload` | Enables diagnostic payload logging where supported. |
| `st_inputPayload` | Preserves the original receiver input payload where needed for diagnostics. |
| `st_idempotencyRequestId` | Stable request identity across retries and manual recovery. |
| `st_idempotencyFirstSent` | Original first-sent timestamp for applicable OData V4 repeatability handling. |

**Important:** the actual router determines the retry counter, delay/backoff, and how `st_autoRetry` is read. Do not assume that setting a header alone schedules a retry.

## 6. Receiver iFlow Processing Pattern

A typical receiver iFlow follows these steps:

1. **Initialize receiver context:** set receiver and message-type headers, retry settings, and any necessary correlation identifiers.
2. **Transform the message:** map source data into the target API or RFC structure.
3. **Support testing and diagnostics:** optionally log the transformed payload or stop before the backend call in testing mode.
4. **Invoke the target:** call the receiver using the configured adapter.
5. **Validate the result:** inspect HTTP status, API error payload, BAPI `RETURN`, or other business-specific response details.
6. **On failure:** populate framework diagnostics and propagate the failure to the global router.
7. **On success:** return normally so the router can acknowledge successful processing.

### 6.1 Response validation

An HTTP success code does not always imply business success. For example, an RFC BAPI can return an error in its `RETURN` structure, and an OData response may contain application-level error details.

Validation scripts should:

- Extract the most useful error code and message.
- Distinguish retryable and non-retryable conditions.
- Set `st_errorMessage`, `st_errorDetails`, and `st_autoRetry` consistently.
- Throw or rethrow an exception for failures so the global router can handle them.

Avoid swallowing an exception and returning a normal success result when the receiver operation has failed.

## 7. Automatic Retry Handling

The global router should apply a bounded retry policy:

```text
Receiver exception
       |
       v
Classify error
       |
       v
Retry allowed AND safe?
    /             \
  Yes              No
   |                |
Retry count       Error Data Store
below limit?      / manual recovery
  /      \
Yes      No
 |        |
Retry    Error Data Store
```

Recommended principles:

- Apply a finite maximum number of attempts.
- Consider delays/backoff for transient failures and rate limiting.
- Keep correlation and idempotency identifiers stable.
- Do not retry business validation failures by default.
- Record the final error and relevant context after retries are exhausted.
- Ensure the JMS transaction and router exception path do not accidentally acknowledge and lose a failed message.

## 8. Error Data Store and Manual Recovery

When the router determines that a message should no longer be retried automatically, the framework can persist the failed message in a receiver error Data Store, using the `STerr_` prefix (for example, `STerr_SAPS4H`).

A recoverable error record should retain, as supported by the implementation:

- Original business payload or an appropriate recoverable copy
- Interface, sender, and receiver context
- Message/correlation identifiers
- Error code, message, and technical details
- Retry status and relevant timestamps

**Recovery workflow:**

1. Locate the failed message using the Message Processing Log and Data Store reference.
2. Identify whether the receiver transaction may already have completed.
3. Correct the payload or underlying configuration/business data as needed.
4. Resubmit using the same logical request identity when duplicate prevention requires it.
5. Verify the backend result and audit the recovery action.

The commercial **ST Integration Toolkit** can provide a user interface for supported Error Data Store operations, but it is not required to implement the underlying framework pattern.

## 9. Idempotency and Duplicate Prevention

Retries can create duplicate business transactions if the original request succeeded but the response was lost.

### 9.1 CPI Idempotent Process Call

Where appropriate, use an **Idempotent Process Call** with a stable message identifier, such as:

```text
${header.st_idempotencyRequestId}
```

If **Skip Process Call for Duplicates** is enabled, subsequent invocations with a completed identifier can be skipped. This only protects operations whose successful completion has been recorded; it does **not** fully protect against the scenario in which the backend committed but Cloud Integration timed out before recording success.

**Critical:** a local exception subprocess must not silently convert a failed receiver call into successful completion inside the idempotent scope.

### 9.2 OData V4 repeatability

For APIs that support repeatability, the receiver can send:

```text
Repeatability-Request-ID: <stable request ID>
Repeatability-First-Sent: <original timestamp>
```

In the OData V4 adapter, include the headers in the configured request-header allowlist, for example:

```text
Repeatability-First-Sent|Repeatability-Request-ID
```

Repeatability behavior is **API-specific**. Confirm that the target API supports these headers and test its behavior for duplicates, in-progress requests, and timeouts. Never regenerate the identifiers when retrying the same logical operation.

## 10. Monitoring and Troubleshooting

Use SAP Cloud Integration Message Processing Logs to identify the failing receiver flow, correlate the message with its original sender, and inspect the error details.

Suggested monitoring fields include:

| Field | Purpose |
|---|---|
| Application Message ID | Business correlation identifier |
| Application Message Type | Logical or technical message type |
| Sender / Receiver | Source and target context |
| `st_errorMessage` | Short human-readable failure summary |
| `st_errorDetails` | Backend diagnostics or Data Store reference |
| Error Data Store entry ID | Manual recovery lookup |

Avoid recording credentials, authorization tokens, personal data, or large payloads in unrestricted logs. Apply access controls and retention policies to Error Data Stores and monitoring artifacts.

## 11. Example Scenarios

| Scenario | Classification | Typical action |
|---|---|---|
| Invalid country code in OData request | Functional | No automatic retry; correct and resubmit |
| Temporary backend lock | Potentially transient | Retry only if configured and safe |
| Receiver HTTP 503 | Technical | Bounded retry if operation is safe to repeat |
| Receiver timeout after POST | Unknown outcome | Reconcile or retry with verified idempotency protection |
| RFC BAPI `RETURN` contains error | Functional or technical, based on code | Validate `RETURN` and apply configured policy |
| Duplicate request already completed | Duplicate | Avoid executing the business operation again; do not automatically treat every duplicate as a new receiver failure |

## 12. Implementation Checklist

- [ ] Receiver iFlow propagates failures to the global router.
- [ ] Retryable and non-retryable error codes are documented.
- [ ] `st_maxRetry` and `st_autoRetry` match the deployed router's configuration model.
- [ ] Retry count is bounded and backoff is considered.
- [ ] Request identity is preserved across retries.
- [ ] Timeout behavior has been tested against the actual backend.
- [ ] Error Data Store records contain enough information for recovery.
- [ ] Logging excludes secrets and respects data-retention requirements.
- [ ] Successful, functional-error, transient-error, and uncertain-outcome scenarios are tested.

## 13. Related Documentation

- [ST Naming Convention](ST_Naming_Convention.md)
- [Sender Exception Handling](Sender_Exception_Handling.md)

## License and Disclaimer

Copyright 2026 ST Consultancy B.V.

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.

This documentation describes a reference pattern; validate the behavior of the specific integration artifacts, adapters, and target APIs before production use. ST Integration Framework is an independent project and is not an official SAP product.
