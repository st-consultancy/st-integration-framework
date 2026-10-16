# ST Integration Framework — Naming Conventions
This document defines recommended naming conventions for SAP Cloud Integration artifacts developed using the **ST Integration Framework**. Consistent names help developers identify an artifact's interface, responsibility, and target without opening its configuration.

## 1. Integration Packages
**Convention:** `<Interface ID> | <Sender System> | <Source Message>`
`<Source Message>` may be a technical IDoc message type, event type, API business entity, or logical message name.

**Examples:**
- `PI0000 | DemoApp | CustomerMsg`
- `PI0001 | SAPECC | ORDERS.ORDERS05`
- `PI0002 | SAPS4H | SalesOrder.Created.v1`

Use a unique package/interface ID for each logical interface, according to your project's interface register.

For shared components, use a descriptive global package name, for example:
- `GLOBAL | Shared Integration Artifacts`

## 2. Interface-Specific Integration Flows
**Convention:** `<Interface ID>_<Role>_<Context>[_<AdditionalSuffix>]`

| Element              | Description |
| -------------------- | ----------- |
| `<Interface ID>`     | Unique identifier, e.g. `IF0000`. |
| `<Role>`             | `Sender`, `Router`, or `Receiver`. |
| `<Context>`          | Role-specific source, message, or target context. |
| `<AdditionalSuffix>` | Optional protocol, variant, or processing purpose. |

**Roles and contexts:**
- **Sender:** Receives or collects messages from a source system. Context identifies the source system or source context.
- **Router:** Performs interface-specific routing, orchestration, splitting, or receiver determination. Context identifies the business object or message type.
- **Receiver:** Processes and delivers messages to a target system. Context identifies the target system.

**Examples:**
- `IF0000_Sender_DemoApp_RFC`
- `IF0000_Sender_DemoApp_CustomerEvents`
- `IF0000_Router_CustomerMsg`
- `IF0000_Receiver_SAPS4H`
- `IF0000_Receiver_SAPS4H_CustomerCreation`

## 3. Global / Shared Integration Flows
**Convention:** `GLOBAL_<Role>_<Context>[_<AdditionalSuffix>]`

**Examples:**
- `GLOBAL_Sender_IDOC`
- `GLOBAL_Receiver_SAPS4H`

## 4. Message Mappings
**Convention:** `MM_<SourceMessage>_to_<TargetMessage>`

**Examples:**
- `MM_SHPMNT05_to_Shipment`
- `MM_ORDERS05_to_SalesOrderRequest`

## 5. XSLT Mappings
Use camelCase file names and distinguish business transformations from technical utilities.

| Purpose | Convention | Examples |
| --- | --- | --- |
| Business transformation | `map<Source>To<Target>.xslt` | `mapSHPMNT05ToShipment.xslt`, `mapSupplierInvoiceToS4Invoice.xslt` |
| Technical transformation | `<verb><Object>.xslt` | `removeNamespaces.xslt`, `removeEmptyElements.xslt`, `sortItems.xslt` |

## 6. ProcessDirect Endpoints
ProcessDirect addresses should mirror their target iFlow names so that the destination can be identified without opening the adapter configuration.

**Convention:** `/<interface-id>/<role>/<context>[/<additional-suffix>]`
Use lowercase path segments. Keep each segment aligned with the corresponding part of the iFlow name.

| Target iFlow | ProcessDirect endpoint |
| --- | --- |
| `IF0000_Receiver_SAPS4H` | `/if0000/receiver/saps4h` |
| `IF0000_Router_Customer` | `/if0000/router/customer` |
| `IF0000_Receiver_SAPS4H_CustomerCreation` | `/if0000/receiver/saps4h/customercreation` |
| `GLOBAL_Receiver_SAPS4H` | `/global/receiver/saps4h` |

## 7. Groovy Scripts and JavaScript Files
**General convention:** `<verb><Object><OptionalContext>.<extension>`
Use camelCase and a descriptive verb-led name.

**Examples:**
- `prepareErrorHandling.groovy`
- `logPayload.groovy`
- `logIDocIDsInCustomHeaders.js`

For shared scripts maintained as part of the ST Integration Framework, use the `st_` prefix to distinguish them from interface-specific scripts:
- `st_prepareErrorHandling.groovy`
- `st_logPayload.groovy`
- `st_logIDocIDsInCustomHeaders.js`

## 8. Framework JMS Queues
**Convention:** `ST_<PurposeOrSystem>`
Use the `ST_` prefix for framework-specific queues to prevent collisions with other teams' queues.

**Examples:**
- `ST_Incoming`
- `ST_SAPS4H`
- `ST_Events`

## 9. Framework Data Stores
| Purpose | Convention | Example |
| --- | --- | --- |
| Sender exception persistence | `STexp_<Purpose>` | `STexp_SenderException` |
| Receiver error persistence   | `STerr_<Purpose>` | `STerr_SAPS4H`          |

Keep Data Store names within SAP Cloud Integration's applicable naming limits (40 characters for the framework's Data Store naming standard).

## 10. Framework Headers and Exchange Properties
**Convention:** `st_<camelCase>`
Use the `st_` prefix for framework-specific message headers and exchange properties to avoid conflicts with SAP/Camel standard fields and interface-specific attributes.

**Examples:**
- `st_autoRetry`
- `st_maxRetry`
- `st_errorMessage`
- `st_errorDetails`
- `st_logPayload`
- `st_idempotencyRequestId`
- `st_idempotencyFirstSent`

Whether a value is a **header** or **exchange property** depends on its purpose and the framework component consuming it; the prefix alone does not determine its storage location.

Maintained by **ST Consultancy B.V.**  
See the repository's `LICENSE` file for licensing terms.
