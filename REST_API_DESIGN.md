# Imara Financial Services REST API Design Blueprint

## Executive Summary

This document proposes a REST API design blueprint for Imara Financial Services, an East African merchant credit and remittance platform. The first slice of the API focuses on registering merchants, supporting field-agent onboarding workflows, capturing financing requests, connecting those requests to lending partners, tracking disbursements, and recording basic merchant remittance transactions.

The design emphasizes clear resources, predictable endpoint naming, consistent JSON response structures, realistic validation rules, and business constraints that matter in a financial services environment. The API is design-only at this stage and is intended to guide later work on implementation, validation, authentication, testing, and release readiness.

## Business and Domain Analysis

### Main Actors

| Actor | Role in the Platform | API Needs |
|---|---|---|
| Merchant | Small business owner using Imara for credit or remittance services | Register profile, request financing, view financing status, receive disbursements, send or receive remittances |
| Field Agent | Staff member or representative who helps onboard merchants and verify information | Create merchant records, update verification status, submit supporting notes |
| Lending Partner | External financial partner that reviews and funds financing requests | View eligible financing requests, approve or reject requests, confirm disbursement |
| Imara Operations Team | Internal team responsible for monitoring risk, compliance, and platform activity | Review merchant data, track request status, investigate failed disbursements |

### Key Workflows

1. A field agent registers a merchant and records basic business information.
2. The merchant profile is reviewed and assigned a verification status.
3. A merchant submits a financing request.
4. A lending partner reviews the request and makes a decision.
5. If approved, a disbursement record is created and tracked.
6. A merchant sends or receives a remittance transaction through the platform.
7. The platform records transaction status changes so failed, completed, or reversed payments can be traced.

### Essential Data

| Data Category | Essential Fields | Optional Fields at This Stage |
|---|---|---|
| Merchant identity | Merchant name, phone number, country, business type, registration date, verification status | Email, tax identification number, address details |
| Agent activity | Agent name, phone number, assigned region, active status | Supervisor, branch office |
| Financing request | Merchant ID, amount requested, currency, purpose, request status, submitted date | Supporting documents, risk notes |
| Lending partner | Partner name, supported countries, supported currencies, active status | Contact person, settlement account details |
| Disbursement | Financing request ID, partner ID, amount, currency, status, payment reference | Failure reason, retry count |
| Remittance transaction | Merchant ID, sender or receiver details, amount, currency, destination country, transaction status | External provider reference, failure reason |

### Real-World Constraints

The API design should reflect financial and operational constraints from the start:

- Phone numbers should be unique where they identify merchants or agents.
- Currency values must be explicit because Imara may operate across multiple East African markets.
- Financing requests should only be created for verified or reviewable merchants, depending on the business rule chosen.
- Disbursement records should not exist without an approved financing request.
- Remittance transactions should clearly show direction, currency, and status because they may involve more than one country or payment network.
- Status changes should be controlled so that records do not move into impossible states.
- The API should return clear validation and business-rule errors because financial workflows require traceability.

## Resource Specifications

### 1. Merchants

The `Merchant` resource represents a small business using Imara services.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique merchant identifier, such as `mch_001` |
| businessName | string | Yes | Registered or trading name of the merchant |
| ownerName | string | Yes | Main contact or business owner |
| phoneNumber | string | Yes | Merchant phone number in international format |
| country | string | Yes | Country where the merchant operates |
| businessType | string | Yes | Type of business, such as retail, food, or services |
| verificationStatus | string | Yes | `pending`, `verified`, `rejected`, or `suspended` |
| createdAt | string | Yes | ISO 8601 timestamp when the merchant was created |
| agentId | string | No | Field agent who onboarded the merchant |

Key relationships:

- A merchant may be onboarded by one field agent.
- A merchant may have many financing requests.
- A merchant may receive many disbursements through approved financing requests.

Business rules:

- `phoneNumber` must be unique for active merchants.
- `verificationStatus` must use an approved status value.
- A suspended merchant cannot create a new financing request.

### 2. Field Agents

The `FieldAgent` resource represents a person responsible for onboarding and supporting merchants.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique field agent identifier, such as `agt_001` |
| fullName | string | Yes | Agent's full name |
| phoneNumber | string | Yes | Agent phone number |
| region | string | Yes | Assigned operating region |
| active | boolean | Yes | Whether the agent can currently onboard merchants |
| createdAt | string | Yes | ISO 8601 timestamp when the agent was created |

Key relationships:

- A field agent can onboard many merchants.

Business rules:

- Inactive agents cannot be assigned to new merchant records.
- Agent phone numbers should be unique.

### 3. Financing Requests

The `FinancingRequest` resource represents a merchant's request for credit.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique financing request identifier, such as `fin_001` |
| merchantId | string | Yes | ID of the merchant requesting financing |
| amount | number | Yes | Requested amount |
| currency | string | Yes | ISO-style currency code such as `RWF`, `KES`, `TZS`, or `UGX` |
| purpose | string | Yes | Reason for financing request |
| status | string | Yes | `submitted`, `under_review`, `approved`, `rejected`, `cancelled`, or `disbursed` |
| submittedAt | string | Yes | ISO 8601 timestamp when request was submitted |
| reviewedAt | string | No | ISO 8601 timestamp when partner or operations team reviewed request |
| partnerId | string | No | Lending partner assigned to review or fund the request |

Key relationships:

- A financing request belongs to one merchant.
- A financing request may be reviewed by one lending partner.
- An approved financing request may have one or more disbursement records if partial disbursement is supported.

Business rules:

- `amount` must be greater than zero.
- `currency` must be supported by Imara and the selected lending partner.
- A suspended merchant cannot submit a financing request.
- A request cannot be disbursed unless it has already been approved.

### 4. Lending Partners

The `LendingPartner` resource represents an external partner that can review or fund financing requests.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique partner identifier, such as `ptn_001` |
| name | string | Yes | Lending partner name |
| supportedCountries | array of strings | Yes | Countries where the partner can operate |
| supportedCurrencies | array of strings | Yes | Currencies the partner can fund |
| active | boolean | Yes | Whether the partner is currently available |
| createdAt | string | Yes | ISO 8601 timestamp when the partner was created |

Key relationships:

- A lending partner can review many financing requests.
- A lending partner can be linked to many disbursements.

Business rules:

- Inactive partners cannot be assigned to new financing requests.
- A partner should only receive requests from supported countries and currencies.

### 5. Disbursements

The `Disbursement` resource tracks movement of approved funds to a merchant.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique disbursement identifier, such as `dsb_001` |
| financingRequestId | string | Yes | Approved financing request being funded |
| merchantId | string | Yes | Merchant receiving funds |
| partnerId | string | Yes | Lending partner funding the disbursement |
| amount | number | Yes | Amount disbursed |
| currency | string | Yes | Currency used for disbursement |
| status | string | Yes | `pending`, `processing`, `completed`, `failed`, or `reversed` |
| paymentReference | string | No | External payment or mobile money reference |
| createdAt | string | Yes | ISO 8601 timestamp when disbursement was created |
| completedAt | string | No | ISO 8601 timestamp when disbursement completed |

Key relationships:

- A disbursement belongs to one approved financing request.
- A disbursement is funded by one lending partner.
- A disbursement is received by one merchant.

Business rules:

- Disbursements can only be created for approved financing requests.
- `amount` cannot exceed the approved financing amount.
- Completed disbursements should not be edited except through reversal or adjustment workflows.

### 6. Remittance Transactions

The `RemittanceTransaction` resource records money transfers connected to merchant activity.

| Field | Type | Required | Description |
|---|---|---:|---|
| id | string | Yes | Unique remittance identifier, such as `rmt_001` |
| merchantId | string | Yes | Merchant sending or receiving the transfer |
| direction | string | Yes | `inbound` or `outbound` |
| counterpartyName | string | Yes | Sender or receiver name on the other side of the transaction |
| counterpartyPhone | string | No | Sender or receiver phone number |
| amount | number | Yes | Transfer amount |
| currency | string | Yes | Currency used for the transfer |
| destinationCountry | string | Yes | Country where funds are being sent or received |
| status | string | Yes | `pending`, `processing`, `completed`, `failed`, or `reversed` |
| providerReference | string | No | External payment network or mobile money reference |
| createdAt | string | Yes | ISO 8601 timestamp when the transaction was created |
| completedAt | string | No | ISO 8601 timestamp when the transaction completed |

Key relationships:

- A remittance transaction belongs to one merchant.
- A merchant may have many remittance transactions.

Business rules:

- `amount` must be greater than zero.
- `direction` must be either `inbound` or `outbound`.
- Completed transactions should only be changed through reversal or dispute workflows.
- The selected `currency` and `destinationCountry` must be supported by the platform.

## Endpoint Map

### Merchants

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Merchants | List merchants | GET | `/api/v1/merchants` | Query: `?country=Rwanda&verificationStatus=verified` | `200 OK` with merchant list | `400 Bad Request` for invalid query value |
| Merchants | Create merchant | POST | `/api/v1/merchants` | `{ "businessName": "Amina Shop", "ownerName": "Amina N.", "phoneNumber": "+250788123456", "country": "Rwanda", "businessType": "retail", "agentId": "agt_001" }` | `201 Created` with merchant object | `409 Conflict` if phone number already exists |
| Merchants | Get merchant | GET | `/api/v1/merchants/{merchantId}` | None | `200 OK` with merchant object | `404 Not Found` if merchant does not exist |
| Merchants | Update merchant | PATCH | `/api/v1/merchants/{merchantId}` | `{ "businessType": "food", "verificationStatus": "verified" }` | `200 OK` with updated merchant | `422 Unprocessable Entity` for invalid status transition |

### Field Agents

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Field Agents | List agents | GET | `/api/v1/field-agents` | Query: `?region=Kigali&active=true` | `200 OK` with agent list | `400 Bad Request` for invalid query value |
| Field Agents | Create agent | POST | `/api/v1/field-agents` | `{ "fullName": "Jean M.", "phoneNumber": "+250788111222", "region": "Kigali", "active": true }` | `201 Created` with agent object | `409 Conflict` if phone number already exists |
| Field Agents | Get agent | GET | `/api/v1/field-agents/{agentId}` | None | `200 OK` with agent object | `404 Not Found` if agent does not exist |
| Field Agents | Update agent | PATCH | `/api/v1/field-agents/{agentId}` | `{ "active": false }` | `200 OK` with updated agent | `422 Unprocessable Entity` if update violates assignment rule |

### Financing Requests

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Financing Requests | List requests | GET | `/api/v1/financing-requests` | Query: `?status=submitted&currency=RWF` | `200 OK` with request list | `400 Bad Request` for invalid status |
| Financing Requests | Create request | POST | `/api/v1/financing-requests` | `{ "merchantId": "mch_001", "amount": 500000, "currency": "RWF", "purpose": "Inventory restock" }` | `201 Created` with request object | `422 Unprocessable Entity` if merchant is suspended |
| Financing Requests | Get request | GET | `/api/v1/financing-requests/{requestId}` | None | `200 OK` with request object | `404 Not Found` if request does not exist |
| Financing Requests | Review request | PATCH | `/api/v1/financing-requests/{requestId}` | `{ "status": "approved", "partnerId": "ptn_001" }` | `200 OK` with updated request | `422 Unprocessable Entity` for invalid status transition |

### Lending Partners

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Lending Partners | List partners | GET | `/api/v1/lending-partners` | Query: `?active=true&currency=RWF` | `200 OK` with partner list | `400 Bad Request` for unsupported query |
| Lending Partners | Create partner | POST | `/api/v1/lending-partners` | `{ "name": "Lake Capital", "supportedCountries": ["Rwanda", "Kenya"], "supportedCurrencies": ["RWF", "KES"], "active": true }` | `201 Created` with partner object | `409 Conflict` if partner name already exists |
| Lending Partners | Get partner | GET | `/api/v1/lending-partners/{partnerId}` | None | `200 OK` with partner object | `404 Not Found` if partner does not exist |
| Lending Partners | Update partner | PATCH | `/api/v1/lending-partners/{partnerId}` | `{ "active": false }` | `200 OK` with updated partner | `422 Unprocessable Entity` if partner has pending disbursements |

### Disbursements

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Disbursements | List disbursements | GET | `/api/v1/disbursements` | Query: `?status=completed&merchantId=mch_001` | `200 OK` with disbursement list | `400 Bad Request` for invalid status |
| Disbursements | Create disbursement | POST | `/api/v1/disbursements` | `{ "financingRequestId": "fin_001", "partnerId": "ptn_001", "amount": 500000, "currency": "RWF" }` | `201 Created` with disbursement object | `422 Unprocessable Entity` if request is not approved |
| Disbursements | Get disbursement | GET | `/api/v1/disbursements/{disbursementId}` | None | `200 OK` with disbursement object | `404 Not Found` if disbursement does not exist |
| Disbursements | Update status | PATCH | `/api/v1/disbursements/{disbursementId}` | `{ "status": "completed", "paymentReference": "MOMO-2026-001" }` | `200 OK` with updated disbursement | `422 Unprocessable Entity` for invalid status transition |

### Remittance Transactions

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Remittance Transactions | List transactions | GET | `/api/v1/remittance-transactions` | Query: `?merchantId=mch_001&status=completed` | `200 OK` with transaction list | `400 Bad Request` for invalid status |
| Remittance Transactions | Create transaction | POST | `/api/v1/remittance-transactions` | `{ "merchantId": "mch_001", "direction": "outbound", "counterpartyName": "David K.", "counterpartyPhone": "+254700111222", "amount": 120000, "currency": "RWF", "destinationCountry": "Kenya" }` | `201 Created` with transaction object | `422 Unprocessable Entity` if currency or destination is unsupported |
| Remittance Transactions | Get transaction | GET | `/api/v1/remittance-transactions/{transactionId}` | None | `200 OK` with transaction object | `404 Not Found` if transaction does not exist |
| Remittance Transactions | Update status | PATCH | `/api/v1/remittance-transactions/{transactionId}` | `{ "status": "completed", "providerReference": "MM-908771" }` | `200 OK` with updated transaction | `422 Unprocessable Entity` for invalid status transition |

## Validation and Error Strategy

The API should use consistent JSON responses for validation errors, missing resources, conflicts, and business-rule failures. This makes the API easier for frontend applications, partner systems, and test suites to consume.

### Standard Error Shape

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields are invalid.",
    "details": [
      {
        "field": "amount",
        "message": "Amount must be greater than zero."
      }
    ]
  }
}
```

### Validation and Business-Rule Checks

| Check | Example Failure | Status Code | Reason |
|---|---|---:|---|
| Required fields | Creating a merchant without `businessName` | `400 Bad Request` | The request body is incomplete |
| Duplicate unique field | Creating two merchants with the same `phoneNumber` | `409 Conflict` | The request conflicts with an existing record |
| Invalid enum value | Setting financing status to `paid_out` instead of an allowed status | `400 Bad Request` | The value does not match the API contract |
| Invalid business rule | Creating a financing request for a suspended merchant | `422 Unprocessable Entity` | The JSON may be valid, but the action is not allowed |
| Unsupported transfer corridor | Creating a remittance from Rwanda to a country Imara does not support | `422 Unprocessable Entity` | The request is structurally valid, but the route is not available |
| Missing resource | Requesting `/api/v1/merchants/mch_999` when it does not exist | `404 Not Found` | The target resource cannot be found |
| Invalid status transition | Moving a request from `rejected` to `disbursed` | `422 Unprocessable Entity` | The requested change breaks the workflow |

### Example Validation Failure

Request:

```http
POST /api/v1/financing-requests
Content-Type: application/json
```

```json
{
  "merchantId": "mch_001",
  "amount": -5000,
  "currency": "RWF",
  "purpose": "Inventory restock"
}
```

Response:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields are invalid.",
    "details": [
      {
        "field": "amount",
        "message": "Amount must be greater than zero."
      }
    ]
  }
}
```

### Example Business-Rule Failure

```json
{
  "error": {
    "code": "BUSINESS_RULE_VIOLATION",
    "message": "A disbursement can only be created for an approved financing request.",
    "details": [
      {
        "field": "financingRequestId",
        "message": "The referenced financing request is currently under_review."
      }
    ]
  }
}
```

## Trade-off Rationale

### Decision 1: Separate Disbursements from Financing Requests

I chose to model `Disbursement` as a separate resource instead of storing disbursement details directly inside `FinancingRequest`.

The benefit is that financing approval and payment movement are related but different business events. A request can be approved before money is actually sent, and a disbursement can fail, be retried, or be reversed. Keeping disbursements separate makes those states easier to track.

The trade-off is that developers must join or fetch more than one resource to see the full financing lifecycle. This adds some complexity for simple screens.

This trade-off is reasonable because financial platforms need clear audit trails. Separating approval from fund movement creates a cleaner design for later testing, reporting, and compliance work.

### Decision 2: Use PATCH for Status Updates

I chose to use `PATCH` for updates such as merchant verification, financing review, and disbursement status changes.

The benefit is that clients can update only the fields that changed instead of replacing an entire resource. This fits workflows where only a status or review field changes.

The trade-off is that `PATCH` requires careful validation, especially around allowed status transitions. Without clear rules, clients might attempt incomplete or invalid updates.

This trade-off is still reasonable because the API can enforce status-transition rules and return clear `422 Unprocessable Entity` responses when a requested change violates the workflow.

## Final Reflection

<!-- Write this section yourself before submitting. Suggested prompts:
- What part of the design was hardest for you to decide?
- Which resource or endpoint are you most confident about?
- What would you improve if this API moved into implementation next week?
- How did you make sure the design fits the Imara business context?

Insights to focus on:
- Mention that the platform is not only about storing data; it has real financial workflow states such as pending, approved, completed, failed, and reversed.
- Explain why you separated credit-related resources from remittance transactions.
- Point out that validation matters because bad data in a financial API can affect merchants, partners, and audit trails.
- Reflect on one design decision you personally agreed with or changed after reviewing the draft.
-->

## AI Use Appendix

AI tool used: GPT-5.5 through ChatGPT/Codex.

What I asked it to help with: I asked GPT-5.5 questions to better understand the project context, the assignment goals, and how a REST API blueprint should connect business actors, resources, endpoints, validation rules, and trade-off reasoning.

Ideas I kept: I kept the main REST resource structure, the consistent endpoint table format, the standard JSON error shape, and the idea of separating merchants, field agents, financing requests, lending partners, disbursements, and remittance transactions into different resources.

Ideas I changed or rejected: I reviewed the suggested structure and adjusted the design so it covered both merchant credit and remittance activity, not only loans. I also kept the design at the blueprint level instead of adding implementation code, because the assignment asks for API design only.

Why the final design reflects my own judgment: I used GPT-5.5 for clarification and organization, but I reviewed the assignment brief and chose the final resources, workflows, validation rules, and trade-offs based on what I understood Imara would need in an early API design.
