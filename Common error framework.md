
# Common Error Framework  
SAP PI/PO & MuleSoft – Unified Error Handling via JDBC

## 1. Overview
This document defines a Common Error Framework used by SAP PI/PO and MuleSoft to standardize error handling, persistence, reporting, and troubleshooting across middleware platforms.

Both platforms map native errors into a Canonical Error Envelope and persist them into a shared database using JDBC.

### Objectives
- Eliminate platform-specific error silos
- Enable centralized monitoring & analytics
- Support faster triage and root-cause analysis
- Provide consistent error taxonomy and governance

---

## 2. High-Level Architecture
**Error Flow**
1. Error occurs in SAP PI/PO or MuleSoft  
2. Native error is mapped to Canonical Error Envelope  
3. Canonical error is inserted into a shared DB via JDBC  
4. Dashboards, alerts, and BI tools consume the DB  

---

## 3. Canonical Error Envelope
One error equals one record, with consistent identifiers across platforms.

### Standard Fields
- errorId (UUID)
- sourceSystem (SAP_PIPO | MULESOFT)
- environment (DEV | TEST | PROD)
- integrationName
- timestampUtc
- severity (INFO | WARN | ERROR | CRITICAL)
- category (TECHNICAL | BUSINESS)
- errorCode (E-000X)
- message
- details
- correlationId
- transactionId
- endpoint
- retryable
- retryCount
- payloadRef
- tags
- rawError

---

## 4. Canonical JSON Example (MuleSoft)
```json
{
  "errorId": "9f6b8c1e-8d11-4b6f-9e8f-1c4a0a2f1a9c",
  "sourceSystem": "MULESOFT",
  "environment": "PROD",
  "integrationName": "CustomerUpsert",
  "timestampUtc": "2025-12-23T13:15:22Z",
  "severity": "ERROR",
  "category": "TECHNICAL",
  "errorCode": "E-0001",
  "message": "Database connection timed out",
  "details": {
    "exceptionType": "SQLTimeoutException",
    "rootCause": "Connection pool exhausted"
  },
  "correlationId": "CORR-8b3e2f1a9c0d4f33",
  "transactionId": "mule-msg-01JFX2K3N9QZ",
  "endpoint": "jdbc://core-integration-db",
  "retryable": true,
  "retryCount": 2,
  "payloadRef": "s3://integration-payloads/prod/customer/9f6b8c1e.json",
  "tags": ["jdbc", "timeout"],
  "rawError": {
    "muleErrorType": "DB:CONNECTIVITY",
    "description": "Timeout acquiring a connection"
  }
}
```

---

## 5. Canonical XML Example (SAP PI/PO)
```xml
<ErrorEnvelope>
  <errorId>0b73e1b3-4bf2-4f7a-8f8a-bc9f2b20df19</errorId>
  <sourceSystem>SAP_PIPO</sourceSystem>
  <environment>PROD</environment>
  <integrationName>ORDERS_OUTBOUND_IF</integrationName>
  <timestampUtc>2025-12-23T13:15:22Z</timestampUtc>
  <severity>ERROR</severity>
  <category>BUSINESS</category>
  <errorCode>E-0004</errorCode>
  <message>Missing required field ShipToCountry</message>
  <details>
    <exceptionType>MappingException</exceptionType>
    <rootCause>Mandatory field missing</rootCause>
  </details>
  <correlationId>CORR-3f0a9c2e11c84a2b</correlationId>
  <transactionId>pipo-msg-5000012345</transactionId>
  <endpoint>Receiver:CRM_SOAP</endpoint>
  <retryable>false</retryable>
  <retryCount>0</retryCount>
  <payloadRef>content-repo://orders/0b73e1b3.xml</payloadRef>
</ErrorEnvelope>
```

---

## 6. Standard Error Code Catalog
| Error Code | Category | Description | Retryable |
|----------|---------|-------------|-----------|
| E-0001 | Technical | Database connectivity failure | Yes |
| E-0002 | Technical | Timeout (API, JDBC, SFTP) | Yes |
| E-0003 | Technical | Authentication / Authorization failure | No |
| E-0004 | Business | Mandatory field missing | No |
| E-0005 | Business | Data validation failed | No |
| E-0006 | Technical | Mapping / transformation failure | No |
| E-0007 | Technical | Target system unavailable | Yes |
| E-0008 | Business | Duplicate record detected | No |
| E-0009 | Technical | File not found / SFTP error | Yes |
| E-0010 | Technical | Unknown / unhandled exception | No |

---

## 7. Database Table
Table: INTEGRATION_ERROR_LOG

Columns:
ERROR_ID, SOURCE_SYSTEM, ENVIRONMENT, INTEGRATION_NAME, TIMESTAMP_UTC,
SEVERITY, CATEGORY, ERROR_CODE, MESSAGE, DETAILS, CORRELATION_ID,
TRANSACTION_ID, ENDPOINT, RETRYABLE, RETRY_COUNT, PAYLOAD_REF, RAW_ERROR, TAGS

---

## 8. Platform Mapping Guidelines
### SAP PI/PO
- transactionId → PI/PO Message ID
- integrationName → Interface / ICO
- rawError → PI/PO error text

### MuleSoft
- transactionId → Mule Message ID
- correlationId → Mule Correlation ID
- rawError → Mule error type and description

---

## 9. Best Practices
- Do not store full payloads in DB
- Always populate correlationId
- Always preserve rawError
- Use standardized error codes only

---

## 10. Benefits
- Unified error visibility
- Faster MTTR
- Platform-agnostic reporting
- Strong governance and scalability
