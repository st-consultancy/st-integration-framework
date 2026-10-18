# Getting Started with ST Integration Framework

## 1. Overview
The **PI0000 | HttpClient | Customer** package provides practical examples of building integrations using the ST Integration Framework on SAP Cloud Integration.

The package demonstrates two common integration patterns:
- **Synchronous Customer Creation:** An HTTP client sends a request to create a Flight Customer in SAP and receives the processing result immediately.
- **Asynchronous Customer Update:** An HTTP client submits a customer change request, which is processed asynchronously using JMS and reusable routing components.

Both scenarios demonstrate the separation of sender and receiver responsibilities, standardized exception handling, and reusable framework components.

The examples use SAP Flight Customer RFC function modules to keep the business logic simple and focus on integration design.

## 2. Integration Architecture

### Scenario 1: Synchronous Customer Creation

![PI0000 Synchronous Integration Architecutre](images/PI0000_architecture_SynchronousCustomerCreation.png)

The synchronous scenario demonstrates how to separate inbound communication from backend processing while maintaining a single request-response interaction.

### Scenario 2: Asynchronous Customer Update

![PI0000 Asynchronous Integration Architecutre](images/PI0000_architecture_AsynchronousCustomerChange.png)

The asynchronous scenario decouples message reception from backend processing.

Once a message is successfully handed over to JMS, the sender can acknowledge the request without waiting for SAP backend processing to complete.

## 3. Integration Flow Reference

### IF0000_Sender_HttpClient_createCustomerSync
**Purpose:** Receives synchronous customer creation requests via HTTPS and forwards them to the receiver iFlow using ProcessDirect.

![Synchronous Customer Creation - Sender iFlow](images/IF0000_Sender_HttpClient_createCustomerSync.png)

**Processing Steps**
1. **Assign Headers:** Initializes the framework headers and properties required for message processing.
2. **Call Receiver iFlow:** Invokes `IF0000_Receiver_SAPS4H_createFlightCustomer` synchronously using ProcessDirect.
3. **Set Response Header:** Prepares the HTTP response headers before returning the processing result to the client.

**Standardized Sender Exception Handling**
If an exception occurs, the iFlow invokes the reusable sender exception-handling process, which performs the following steps:
- **e1. Prepare Error Log:** Sets the target exception data store `st_exceptionDataStore` and collects exception details for troubleshooting.
- **e2. Add Custom Headers:** Adds relevant metadata for monitoring and investigation.
- **e3. Save to Data Store:** Persists exception details in the sender exception Data Store (`STexp_...`).
- **e4. Set Sender Response:** Prepares an appropriate error response for the HTTP client.

The exception-handling process also includes a **fallback mechanism (e5)** to return an error response if the exception handler itself fails.

### IF0000_Receiver_SAPS4H_createFlightCustomer
**Purpose:** Creates a Flight Customer in SAP using RFC and returns the processing result to the sender iFlow synchronously.

![Synchronous Customer Creation - Receiver iFlow](images/IF0000_Receiver_SAPS4H_createFlightCustomer.png)

**Processing Steps**
1. **Assign Headers:** Initializes the required headers for message processing.
2. **Map XML to RFC:** Transforms the incoming customer XML into the request structure expected by the SAP RFC function module.
3. **Call RFC:** Invokes the SAP backend through the RFC receiver adapter to create the Flight Customer.
4. **Validate BAPI Return:** Evaluates the BAPI return messages to identify functional errors that may not trigger an RFC adapter exception.
5. **Map BAPI to XML:** Transforms the SAP response into the XML format returned to the sender iFlow.

**Error Handling**
The receiver iFlow distinguishes between technical exceptions and functional errors:
- **Technical exceptions:** RFC connectivity failures, timeouts, or other runtime exceptions propagate to the synchronous sender iFlow.
- **Functional errors:** The BAPI return validation step checks for application-level errors and can raise an exception when customer creation fails.

In both cases, the sender iFlow handles the propagated exception through the framework's standardized sender exception-handling process.

### IF0000_Sender_HttpClient_changeCustomerAsync
**Purpose:** Receives asynchronous customer update requests via HTTPS and publishes them to the `ST_SAPS4H` JMS queue for processing by the receiver iFlow.

![Asynchronous Customer Update - Sender iFlow](images/IF0000_Sender_HttpClient_changeCustomerAsync.png)

**Processing Steps**
1. **Assign Headers:** Initializes the framework headers and properties, including `st_mainFlow`, which identifies the receiver iFlow to be invoked by the global router.
2. **Simulate JMS Failure:** Provides a testing mechanism to simulate a JMS failure and demonstrate standardized sender exception handling.
3. **Save to Queue:** Publishes the customer update message to the `ST_SAPS4H` JMS queue for asynchronous processing.
4. **Set Response Header:** Prepares the HTTP response headers to acknowledge successful message submission.

**Standardized Sender Exception Handling**
If an exception occurs during message reception or JMS publishing, the iFlow invokes the reusable sender exception-handling process:
- **e1. Prepare Error Payload:** Sets the target exception data store `st_exceptionDataStore`, collects exception details and prepares the error payload.
- **e2. Add Custom Headers:** Adds relevant metadata for monitoring and troubleshooting.
- **e3. Save to Data Store:** Persists exception details in the sender exception Data Store (`STexp_...`) for investigation.
- **e4. Set Sender Response:** Prepares an appropriate HTTP error response.

The exception-handling process also includes a **fallback mechanism (e5)** to return an error response if the exception handler itself fails.

**Important:** A successful HTTP response confirms that the message has been handed over to JMS. It does not indicate that the customer update has been completed successfully in SAP. Backend processing and receiver error handling are performed independently by `ST_Router_SAPS4H` and the receiver iFlow.

### IF0000_Receiver_SAPS4H_changeFlightCustomer
**Purpose:** Processes asynchronous customer update requests received from `ST_Router_SAPS4H` via ProcessDirect and updates Flight Customer data in SAP using RFC.

![Asynchronous Customer Update - Receiver iFlow](images/IF0000_Receiver_SAPS4H_changeFlightCustomer.png)

**Processing Steps**
1. **Set Headers & Properties:** Initializes the framework headers and properties required for message processing and error handling.
2. **Map XML to RFC:** Transforms the incoming customer XML into the request structure expected by the SAP RFC function module.
3. **Call S/4HANA:** Invokes the SAP backend through the RFC receiver adapter to update the Flight Customer.
4. **Handle RFC Response:** Evaluates the RFC response and handles application-level errors that may not trigger a technical exception.

**Standardized Receiver Error Handling**
Unlike the synchronous scenario, asynchronous processing does not return backend errors directly to the original HTTP client. Instead, the framework handles processing failures through the reusable `ST_Router_SAPS4H` integration flow.

Depending on the error type and framework configuration:
- **Automatic Retry:** Transient errors can trigger configurable retry processing.
- **Error Persistence:** Failed messages can be stored in the receiver Error Data Store (`STerr_...`) for investigation and recovery.
- **Manual Recovery:** Failed messages can be reprocessed using the `ST_Retry_FailedMessage` iFlow.

This separation allows receiver iFlows to focus on backend communication and business processing while the framework provides standardized error handling.

**Important:** The receiver iFlow does not implement its own JMS retry mechanism. Retry and error persistence are managed by the reusable routing framework, allowing consistent error handling across multiple receiver iFlows.

## 4. Understanding Error Handling
One of the key design principles of ST Integration Framework is the distinction between **sender exceptions** and **receiver errors**.

|                        | Sender exception                                    | Receiver error                             |
|------------------------|-----------------------------------------------------|--------------------------------------------|
| Occurs during          | Message reception or synchronous request processing | Asynchronous backend processing            |
| Primary responsibility | Inform the sender and support investigation         | Support retry, investigation, and recovery |
| Data Store prefix      | `STexp_`                                            | `STerr_`                                   |
| Automatic retry        | Not the primary mechanism                           | Supported where configured                 |
| Message recovery       | Investigation-focused                               | Supports reprocessing                      |

### Sender Exception Handling
Sender exceptions occur when a request cannot be processed successfully during its initial communication flow.

The framework captures relevant error details and can persist exception information in a Data Store using the `STexp_` prefix.

These entries are intended primarily for troubleshooting and investigation.

### Receiver Error Handling
Receiver errors occur when an asynchronous message cannot be processed successfully by the target system.

Depending on the configuration and error type, the framework can retry processing automatically.

Messages requiring investigation or manual recovery can be stored in the Error Data Store using the `STerr_` prefix.

## 5. Failed-Message Recovery
The accelerator package provides a reusable recovery iFlow:

**`ST_Retry_FailedMessage`**
This iFlow allows developers or operators to reprocess a previously failed message without requiring the commercial ST Integration Toolkit.

The recovery process is:
1. Identify the failed message in the Error Data Store.
2. Configure `st_errorDataStoreName` and `st_errorDataStoreEntryID`.
3. Execute `ST_Retry_FailedMessage`.
4. The iFlow retrieves the stored message and republishes it to `ST_Incoming`.
5. `ST_Router_Incoming` dispatches the message to the appropriate receiver using `st_mainFlow`.

Successful republication to JMS does not necessarily mean that backend processing has completed successfully.

## 6. Prerequisites
Before deploying the examples, ensure the following are available:
- SAP Integration Suite — Cloud Integration
- ST Integration Accelerators package
- Required JMS messaging capabilities and queues
- An SAP ABAP backend supporting the Flight Customer RFC function modules
- RFC connectivity to the SAP backend, typically through SAP Cloud Connector
- An HTTP client, such as Postman, for sending test requests

**Note:** Flight Customer RFC function modules are part of SAP's classic Flight Model examples and may not be available in every SAP backend system.

## 7. Installation and Configuration
1. Download the latest release from [GitHub Releases](https://github.com/st-consultancy/st-integration-framework/releases).
2. Import the **ST | Integration Accelerators** package into SAP Cloud Integration.
3. Import the **PI0000 | HttpClient | Customer** package.
4. Configure the required JMS queue and routing settings.
5. Configure the RFC destination and SAP backend connectivity.
6. Configure the HTTPS sender endpoints and required authentication.
7. Deploy the required global router and sample integration flows.
8. Send test requests using an HTTP client.

Refer to the individual iFlow configurations for the required endpoint addresses, payload formats, and externalized parameters.

## 8. Recommended Test Scenarios
| Test                             | Expected behavior                                              |
|----------------------------------|----------------------------------------------------------------|
| Create a customer synchronously  | SAP creates the Flight Customer and returns a response         |
| Trigger a synchronous RFC error  | Sender exception handling prepares an error response           |
| Update a customer asynchronously | Sender accepts the message and hands it over to JMS            |
| Inspect asynchronous routing     | Global router dispatches the message to the receiver iFlow     |
| Trigger a receiver error         | Framework applies configured retry and error-handling behavior |
| Reprocess a failed message       | Recovery iFlow republishes the stored message to JMS           |

## 9. Next Steps
After completing these examples, developers can apply the same integration patterns to their own interfaces.

The framework's reusable components help standardize routing, error handling, retry, and message recovery while keeping interface-specific business logic in dedicated integration flows.

For documentation, bug reports, questions, or suggestions, visit: **[ST Integration Framework on GitHub](https://github.com/st-consultancy/st-integration-framework)**

Developed and maintained by **ST Consultancy B.V.**  
Licensed under **Apache License 2.0**.