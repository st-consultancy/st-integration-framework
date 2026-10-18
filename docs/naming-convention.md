# ST Integration Framework — Naming Conventions

This document defines recommended naming conventions for SAP Cloud Integration artifacts developed using the **ST Integration Framework**.

Consistent naming helps developers and support teams quickly identify an artifact's interface, responsibility, source system, and target system without inspecting its configuration.

These conventions are recommendations rather than mandatory requirements. Existing integrations can adopt the framework without necessarily renaming their artifacts.

## 1. Integration Packages

**Convention:**

`<Interface ID> | <Sender System> | <Source Message>`

The source message can be an IDoc message type, event type, API business entity, or logical message name.

**Examples:**

- `PI0000 | HttpClient | Customer`
- `PI0001 | SAPECC | ORDERS.ORDERS05`
- `PI0002 | SAPS4H | SalesOrder.Created.v1`

Use a unique interface ID for each logical integration according to your project's interface register.

For shared components, use a descriptive package name, such as:

- `ST | Integration Accelerators`

## 2. Interface-Specific Integration Flows

**Convention:**

`<Interface ID>_<Role>_<Context>[_<AdditionalSuffix>]`

| Element | Description |
|---|---|
| `<Interface ID>` | Unique identifier, such as `IF0000`. |
| `<Role>` | `Sender`, `Router`, or `Receiver`. |
| `<Context>` | Source system, business object, or target system, depending on the role. |
| `<AdditionalSuffix>` | Optional message type, operation, or processing variant. |

### Roles and Responsibilities

- **Sender:** Receives or collects messages from a source system and prepares them for processing.
- **Router:** Performs interface-specific routing, orchestration, splitting, or receiver determination.
- **Receiver:** Processes messages and communicates with the target system.

**Examples:**

- `IF0000_Sender_HttpClient_createCustomerSync`
- `IF0000_Sender_HttpClient_changeCustomerAsync`
- `IF0000_Receiver_SAPS4H_createFlightCustomer`
- `IF0000_Receiver_SAPS4H_changeFlightCustomer`
- `IF0002_Router_SAPS4H`

Use descriptive suffixes when an interface contains multiple operations or processing variants.

## 3. Shared Integration Flows

Reusable framework integration flows use the `ST_` prefix to distinguish them from interface-specific artifacts.

**Convention:**

`ST_<Component>[_<Context>]`

**Examples:**

- `ST_Router_Incoming`
- `ST_Router_SAPS4H`
- `ST_Router_SAPECC`
- `ST_Collector_Events`
- `ST_Retry_FailedMessage`

Shared integration flows are generally maintained in the **ST | Integration Accelerators** package.

The `ST_` prefix identifies reusable framework components, while `IFxxxx_` identifies interface-specific integration flows.

## 4. Message Mappings

**Convention:**

`MM_<SourceMessage>_to_<TargetMessage>`

**Examples:**

- `MM_SHPMNT05_to_Shipment`
- `MM_ORDERS05_to_SalesOrderRequest`
- `MM_FlightCustomerCreate_to_BAPI_FLCUST_CREATEFROMDATA`

Mapping names should clearly identify the source and target message structures.

## 5. XSLT Mappings

Use camelCase filenames and distinguish business transformations from technical utilities.

| Purpose | Convention | Example |
|---|---|---|
| Business transformation | `map<Source>To<Target>.xslt` | `mapSHPMNT05ToShipment.xslt` |
| Technical transformation | `<verb><Object>.xslt` | `removeNamespaces.xslt` |
| Receiver determination | `<verb><Object>.xslt` | `determineBpReceivers.xslt` |

For reusable framework XSLT mappings, the optional `st_` prefix can be used to distinguish shared utilities from interface-specific mappings.

## 6. ProcessDirect Endpoints

ProcessDirect addresses should be predictable and reflect the target integration flow's interface, role, and processing context.

**Convention:**

`/<interface-id>/<role>/<context>[/<operation>]`

Use lowercase path segments.

**Examples:**

| Target iFlow | ProcessDirect Endpoint |
|---|---|
| `IF0000_Receiver_SAPS4H_createFlightCustomer` | `/if0000/receiver/saps4h/fcust-create` |
| `IF0000_Receiver_SAPS4H_changeFlightCustomer` | `/if0000/receiver/saps4h/fcust-change` |
| `IF0002_Router_SAPS4H` | `/if0002/router/saps4h` |

ProcessDirect addresses are integration contracts between iFlows. Once an endpoint is in use, avoid changing it without updating its consumers.

## 7. Groovy Scripts and JavaScript Files

**General convention:**

`<verb><Object>[<OptionalContext>].<extension>`

Use camelCase and descriptive verb-led names.

**Examples:**

- `prepareErrorHandling.groovy`
- `logPayload.groovy`
- `logIDocIDsInCustomHeaders.js`

For shared scripts maintained as part of the ST Integration Framework, use the `st_` prefix:

- `st_prepareErrorHandling.groovy`
- `st_logPayload.groovy`
- `st_logIDocIDsInCustomHeaders.js`

This helps distinguish reusable framework utilities from interface-specific scripts.

## 8. Framework JMS Queues

**Convention:**

`ST_<PurposeOrSystem>`

Use the `ST_` prefix to distinguish framework queues from other integration queues.

**Examples:**

- `ST_Incoming`
- `ST_SAPS4H`
- `ST_SAPECC`

The queue name typically identifies the processing purpose or target system.

For example:

- `ST_Incoming` — General incoming messages requiring further routing.
- `ST_SAPS4H` — Messages intended for SAP S/4HANA processing.
- `ST_SAPECC` — Messages intended for SAP ECC processing.

## 9. Framework Data Stores

The framework distinguishes between sender exceptions and receiver errors using different Data Store prefixes.

| Purpose | Convention | Example |
|---|---|---|
| Sender exception persistence | `STexp_<Interface>_<Role>_<Context>` | `STexp_IF0000_Sender_createCustomerSync` |
| Receiver error persistence | `STerr_<Interface>_<Role>_<Context>` | `STerr_IF0000_Receiver_changeFlightCustom` |

**Sender Exception Data Stores (`STexp_...`)**

Store diagnostic information for exceptions occurring during message reception or synchronous processing. These entries are intended primarily for investigation and troubleshooting.

**Receiver Error Data Stores (`STerr_...`)**

Store failed asynchronous messages requiring investigation, correction, or manual reprocessing.

Data Store names must respect SAP Cloud Integration naming limits. The framework follows a maximum length of 40 characters, so descriptive suffixes may need to be abbreviated.

## 10. Framework Headers and Exchange Properties

**Convention:**

`st_<camelCase>`

Use the `st_` prefix for framework-specific message headers and exchange properties to distinguish them from SAP/Camel standard fields and interface-specific attributes.

**Examples:**

| Header / Property | Purpose |
|---|---|
| `st_mainFlow` | Identifies the ProcessDirect endpoint for receiver routing. |
| `st_autoRetry` | Controls whether automatic retry is enabled for a processing error. |
| `st_maxRetry` | Defines the maximum number of retry attempts. |
| `st_retryCount` | Tracks the number of retries performed. |
| `st_errorMessage` | Contains the error message for monitoring and troubleshooting. |
| `st_errorDetails` | Provides diagnostic information or references to persisted error details. |
| `st_errorDataStoreName` | Identifies the receiver Error Data Store. |
| `st_errorDataStoreEntryID` | Identifies a failed-message entry for recovery. |
| `st_logPayload` | Controls payload logging behavior. |
| `st_idempotencyRequestId` | Identifies a request for idempotent processing. |
| `st_idempotencyFirstSent` | Records the initial submission time for idempotency handling. |

Whether a value is stored as a **message header** or **exchange property** depends on its purpose and the framework component consuming it. The `st_` prefix does not determine its storage location.

## 11. General Recommendations

- **Be consistent:** Apply the same naming pattern across related interfaces.
- **Use meaningful names:** Names should communicate an artifact's purpose without requiring developers to open it.
- **Avoid unnecessary abbreviations:** Use abbreviations primarily when technical naming limits apply.
- **Keep names stable:** Avoid renaming deployed artifacts or endpoints without considering their dependencies.
- **Separate shared and interface-specific components:** Use `ST_` for reusable framework artifacts and `IFxxxx_` for interface-specific integration flows.
- **Prefer configuration over renaming:** Where possible, use externalized parameters to adapt reusable components to different environments.

---

Maintained by **ST Consultancy B.V.**

For questions or feedback, visit [ST Integration Framework on GitHub](https://github.com/st-consultancy/st-integration-framework).

See the repository's `LICENSE` file for licensing terms.