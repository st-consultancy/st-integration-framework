# ST Integration Framework

**Standardized by Design. Reliable by Default.**

ST Integration Framework is an open-source collection of reusable integration patterns, Groovy scripts, and SAP Cloud Integration artifacts developed by **ST Consultancy B.V.**

The framework helps SAP integration developers build reliable, maintainable, and consistent integrations while reducing repetitive development effort.

It provides reusable capabilities for asynchronous message processing, centralized routing, error handling, automatic retries, and message recovery.

## Key Features

### 1. Standardized Message Processing

Reusable integration patterns for synchronous and asynchronous interfaces, including:

- HTTP, IDoc, RFC, and OData integration scenarios
- JMS-based asynchronous processing
- ProcessDirect communication between integration flows
- Standardized message headers and properties
- Separation of sender, router, and receiver processing

### 2. Centralized Routing

Reusable global router iFlows provide standardized message dispatching and exception handling.

Key capabilities include:

- JMS message consumption
- Dynamic routing to receiver iFlows
- Configurable automatic retry handling
- Centralized processing of receiver exceptions
- Error Data Store integration

### 3. Sender Exception Handling

Standardized exception handling for sender integration flows.

Capabilities include:

- Capturing sender-side processing failures
- Recording diagnostic information in Message Processing Logs
- Persisting failed requests to an exception Data Store where applicable
- Returning structured error responses to HTTP sender applications
- Supporting message recovery using the original request

### 4. Receiver Error Handling

Reusable error handling for receiver integration flows.

Capabilities include:

- Functional and technical error classification
- Configurable retryable error codes
- Automatic retry control
- Timeout handling
- Failed-message persistence
- Diagnostic information collection

### 5. Idempotency and Duplicate Prevention

Reusable patterns for protecting backend systems against duplicate processing.

Capabilities include:

- SAP Cloud Integration Idempotent Process Call
- Request identifiers preserved across retries
- OData V4 repeatability headers where supported
- Duplicate detection and error classification

### 6. Monitoring and Troubleshooting

Standardized monitoring information to simplify troubleshooting.

Capabilities include:

- Application Message ID and Message Type
- Custom Message Processing Log statuses
- Custom headers containing error information
- Data Store references for message recovery
- Diagnostic payload and exception details

## Architecture

The framework separates integration responsibilities into reusable components.

```text
Sender System
     |
     v
Sender iFlow
     |
     v
JMS Queue
     |
     v
Global Router iFlow
     |
     v
Receiver iFlow
     |
     v
Receiver System
```

For synchronous scenarios, sender and receiver iFlows can communicate directly through ProcessDirect without using JMS.

## Framework Components

| Component | Description |
|---|---|
| ST_Collector_Events | Event collection and consolidation |
| ST_Router_Incoming | Global incoming message routing |
| ST_Router_SAPS4H | Routing and processing for SAP S/4HANA |
| ST_Accelerator_Scripts | Reusable Groovy scripts and utilities |

## Example Integration Scenarios

The repository includes demonstration integration packages illustrating how to apply the framework.

| Package | Description |
|---|---|
| PI0001 - BusinessPartnerCreate | Synchronous HTTP-to-OData V2 integration |
| PI0002 - BusinessPartnerCreated | Business Partner to Flight Customer integration |
| PI0003 - FlightCustomerChange | Asynchronous HTTP-to-RFC integration with error handling |
| PI0004 - FlightCustomerEvent | Event-driven integration using SAP Event Mesh |
| PI0005 - FLIGHTCUSTOMER_CHANGE | Asynchronous IDoc-to-OData V4 integration with idempotency |

These examples demonstrate reusable patterns and are intended for learning, testing, and adaptation.

## Getting Started

### Prerequisites

- SAP Integration Suite with Cloud Integration capability
- Access to an SAP BTP subaccount
- Appropriate authorizations to deploy and configure integration artifacts
- SAP backend systems or equivalent test endpoints for the selected demonstration scenario

Additional services, such as JMS and SAP Event Mesh, may be required depending on the integration pattern.

### Installation

1. Download or clone the repository.
2. Import the required integration packages into SAP Cloud Integration.
3. Configure the relevant externalized parameters, destinations, credentials, and endpoints.
4. Deploy the shared framework artifacts.
5. Import and configure a demonstration scenario.
6. Deploy the integration flows and execute the test cases.

Refer to the documentation for each package for configuration details.

## ST Integration Toolkit

ST Integration Framework can be used independently.

For organizations requiring additional operational capabilities, ST Consultancy B.V. also offers the commercial **ST Integration Toolkit**.

The Toolkit provides applications and utilities such as:

- **Error Handling App:** Monitor, correct, retry, and resubmit failed integration messages.
- **Audit Logging App:** Track error-handling and message-recovery activities.
- **iFlow Testing App:** Execute integration tests and compare actual results with expected results.

The ST Integration Toolkit is a separate commercial offering and is not covered by the open-source license of this repository.

## License

ST Integration Framework is licensed under the **Apache License, Version 2.0**.

See [LICENSE](LICENSE) for details.

Copyright 2026 ST Consultancy B.V.

## Contributing

Contributions, bug reports, and suggestions are welcome.

Please use GitHub Issues to report defects or propose improvements.

For code contributions, refer to [CONTRIBUTING.md](CONTRIBUTING.md).

## Disclaimer

This project is provided on an "AS IS" basis, without warranties or conditions of any kind, as described in the Apache License 2.0.

ST Integration Framework is an independent project developed by ST Consultancy B.V. and is not an official SAP product.

SAP product names and trademarks belong to their respective owners.

## About ST Consultancy B.V.

ST Consultancy B.V. is an SAP consulting company specializing in SAP Integration and SAP Business Technology Platform (BTP).

Our expertise includes SAP Cloud Integration, API Management, Event Mesh, SAP BTP application development, and integration architecture.

**Website:** https://st-consultancy.com

**Email:** info@st-consultancy.com

**LinkedIn:** https://linkedin.com/company/st-consultancy88
