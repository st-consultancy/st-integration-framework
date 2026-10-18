# ST Integration Framework

**Standardized by Design. Reliable by Default.**

ST Integration Framework is a free and open-source collection of reusable integration flows, Groovy scripts, and integration patterns for **SAP Cloud Integration**, developed and maintained by **ST Consultancy B.V.**

The framework provides standardized message processing, reusable routing, exception handling, automatic retry, and message recovery capabilities to help developers build reliable and maintainable integrations.

Distributed under the **Apache License 2.0**, the framework can be used, modified, and extended independently, including in commercial projects.

The framework is distributed as **import-ready SAP Cloud Integration packages**. All integration flows and embedded Groovy scripts are included in the downloadable ZIP files and can be inspected and customized directly in SAP Cloud Integration.

## Download

**[Download Latest Release](https://github.com/st-consultancy/st-integration-framework/releases/latest)**

The release includes two import-ready SAP Cloud Integration packages:

| Package | Description |
|---------|-------------|
| **ST \| Integration Accelerators** | Reusable routers, event collector, shared scripts, and message recovery utilities. |
| **PI0000 \| HttpClient \| Customer** | Getting Started examples demonstrating synchronous and asynchronous customer integrations. |

Download and import the packages into your SAP Cloud Integration tenant.

> **Note:** The downloadable CPI package ZIP files are the authoritative distribution of the integration artifacts. This repository does not maintain separate `scripts/` or `src/` directories. Refer to the release assets rather than GitHub's automatically generated source archives.

## Documentation

- **[Getting Started](docs/getting-started.md)** — Installation, configuration, integration scenarios, error handling, and testing examples.
- **[Naming Conventions](docs/Naming_Convention.md)** — Standardized naming conventions for integration packages, iFlows, endpoints, JMS queues, Data Stores, and framework components.

For questions, bug reports, or feature requests, please use [GitHub Issues](https://github.com/st-consultancy/st-integration-framework/issues).

## Key Capabilities

- **Standardized Message Processing:** Reusable patterns for synchronous ProcessDirect and asynchronous JMS-based integrations.
- **Reusable Message Routing:** Centralized routing and dispatching through global router integration flows.
- **Sender Exception Handling:** Consistent error responses, diagnostic logging, and exception persistence for troubleshooting.
- **Receiver Error Handling:** Error classification, configurable automatic retries, failed-message persistence, and manual recovery.
- **Event Collection & Consolidation:** Collection and deduplication of business-object change notifications for efficient downstream processing.
- **Reusable Groovy Scripts:** Shared utilities for message processing, logging, error handling, and integration operations.

The framework promotes the separation of sender, routing, and receiver responsibilities, reducing repetitive development and simplifying integration maintenance.

## Integration Architecture

### Asynchronous Integration

Asynchronous integrations use JMS queues to decouple message reception from backend processing.

Messages are routed through reusable global router iFlows to the appropriate receiver iFlows, where backend processing takes place.

![Asynchronous Integration Architecture](docs/images/ST_IntegrationArchitecture_async.png)

### Synchronous Integration

Synchronous integrations use ProcessDirect communication between sender and receiver iFlows, allowing the processing result to be returned directly to the calling application.

![Synchronous Integration Architecture](docs/images/ST_IntegrationArchitecture_sync.png)

Both patterns follow the framework's standardized approach to message processing and error handling.

## License

ST Integration Framework is licensed under the [Apache License, Version 2.0](LICENSE).

See [NOTICE](NOTICE) for project attribution and notices.

Copyright © 2026 ST Consultancy B.V.

## Feedback & Maintenance

ST Integration Framework is developed and maintained by **ST Consultancy B.V.**

Questions, bug reports, and feature suggestions are welcome through [GitHub Issues](https://github.com/st-consultancy/st-integration-framework/issues).

External code contributions and pull requests are not currently accepted.

## Disclaimer

ST Integration Framework is an independent open-source project. It is not an official SAP product and is not affiliated with or endorsed by SAP SE.

SAP product names and trademarks belong to their respective owners.

The software is provided **AS IS**, without warranties or conditions of any kind, as described in the Apache License 2.0.

## About ST Consultancy B.V.

ST Consultancy B.V. specializes in SAP Integration and SAP Business Technology Platform (BTP), including SAP Cloud Integration, API Management, Event Mesh, and BTP application development.

- **Website:** https://st-consultancy.com
- **Email:** info@st-consultancy.com
- **LinkedIn:** https://linkedin.com/company/st-consultancy88