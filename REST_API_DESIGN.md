# Imara Financial Services REST API Design Blueprint

## Executive Summary

This blueprint designs a first version of a REST API for Imara Financial Services, an East African merchant credit and remittance platform. The API should help Imara onboard merchants, manage field-agent activity, support financing requests, connect approved requests to lending partners, and record payment or remittance activity.

The goal is not to implement the API yet, but to define a clear design that can guide later work on validation, authentication, testing, and release readiness.

## Business and Domain Analysis

The first slice of the platform should focus on four main actors:

| Actor | Main Need |
|---|---|
| Merchant | Register with Imara, request credit, and send or receive money |
| Field Agent | Onboard merchants and help verify merchant details |
| Lending Partner | Review eligible financing requests and fund approved requests |
| Imara Operations Team | Monitor risk, failed payments, status changes, and business rules |

Core workflows:

1. A field agent registers a merchant.
2. Imara verifies or rejects the merchant profile.
3. A merchant submits a financing request.
4. A lending partner reviews and approves or rejects the request.
5. Approved funds are disbursed and tracked.
6. Merchant remittance transactions are recorded with clear status updates.

Important constraints:

- Merchant and agent phone numbers should be unique.
- Amounts must be positive and tied to a clear currency.
- Suspended merchants should not be allowed to create new financing requests.
- Disbursements should only happen after financing approval.
- Payment and request statuses should follow controlled transitions.
- Error responses should be clear because financial workflows need traceability.

## Resource Specifications

### Merchants

Purpose: stores business profile information for merchants using Imara.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique merchant ID |
| businessName | string | Required |
| ownerName | string | Required |
| phoneNumber | string | Required and unique |
| country | string | Required |
| businessType | string | Required |
| verificationStatus | string | `pending`, `verified`, `rejected`, or `suspended` |
| agentId | string | Optional field agent reference |

Relationship: one merchant can have many financing requests and remittance transactions.

Business rule: a suspended merchant cannot create a new financing request.

### Field Agents

Purpose: represents people who onboard and support merchants.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique agent ID |
| fullName | string | Required |
| phoneNumber | string | Required and unique |
| region | string | Required |
| active | boolean | Controls whether the agent can onboard merchants |

Relationship: one field agent can onboard many merchants.

Business rule: inactive agents should not be assigned to new merchant records.

### Financing Requests

Purpose: records a merchant request for credit.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique request ID |
| merchantId | string | Required merchant reference |
| amount | number | Must be greater than zero |
| currency | string | Example: `RWF`, `KES`, `TZS`, `UGX` |
| purpose | string | Required |
| status | string | `submitted`, `under_review`, `approved`, `rejected`, `cancelled`, or `disbursed` |
| partnerId | string | Optional lending partner reference |

Relationship: one merchant can have many financing requests; one partner can review many requests.

Business rule: a financing request cannot be disbursed unless it has been approved.

### Lending Partners

Purpose: represents external partners that fund approved financing requests.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique partner ID |
| name | string | Required and unique |
| supportedCountries | array | Countries the partner supports |
| supportedCurrencies | array | Currencies the partner supports |
| active | boolean | Controls whether partner can receive new requests |

Relationship: one lending partner can review many financing requests and fund many disbursements.

Business rule: inactive partners should not be assigned to new requests.

### Disbursements

Purpose: tracks approved funds moving from a lending partner to a merchant.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique disbursement ID |
| financingRequestId | string | Required approved request reference |
| merchantId | string | Required merchant reference |
| partnerId | string | Required partner reference |
| amount | number | Must not exceed approved amount |
| currency | string | Must match supported currency |
| status | string | `pending`, `processing`, `completed`, `failed`, or `reversed` |
| paymentReference | string | Optional external payment reference |

Business rule: completed disbursements should not be edited directly; reversals should be tracked separately.

### Remittance Transactions

Purpose: records money sent or received through Imara.

| Field | Type | Notes |
|---|---|---|
| id | string | Unique transaction ID |
| merchantId | string | Required merchant reference |
| direction | string | `inbound` or `outbound` |
| counterpartyName | string | Sender or receiver name |
| amount | number | Must be greater than zero |
| currency | string | Required |
| destinationCountry | string | Required |
| status | string | `pending`, `processing`, `completed`, `failed`, or `reversed` |

Relationship: one merchant can have many remittance transactions.

Business rule: unsupported currency or country routes should be rejected.

## Endpoint Map

| Resource | Action | Method | URI | Request Example | Success Response | Error Response |
|---|---|---|---|---|---|---|
| Merchants | Create merchant | POST | `/api/v1/merchants` | `{ "businessName": "Amina Shop", "ownerName": "Amina", "phoneNumber": "+250788123456", "country": "Rwanda", "businessType": "retail" }` | `201 Created` | `409 Conflict` if phone exists |
| Merchants | Get merchant | GET | `/api/v1/merchants/{merchantId}` | None | `200 OK` | `404 Not Found` |
| Merchants | Update verification | PATCH | `/api/v1/merchants/{merchantId}` | `{ "verificationStatus": "verified" }` | `200 OK` | `422 Unprocessable Entity` for invalid status |
| Field Agents | Create agent | POST | `/api/v1/field-agents` | `{ "fullName": "Jean M.", "phoneNumber": "+250788111222", "region": "Kigali", "active": true }` | `201 Created` | `409 Conflict` if phone exists |
| Field Agents | List agents | GET | `/api/v1/field-agents` | Query: `?active=true` | `200 OK` | `400 Bad Request` |
| Financing Requests | Create request | POST | `/api/v1/financing-requests` | `{ "merchantId": "mch_001", "amount": 500000, "currency": "RWF", "purpose": "Inventory" }` | `201 Created` | `422 Unprocessable Entity` if merchant suspended |
| Financing Requests | Review request | PATCH | `/api/v1/financing-requests/{requestId}` | `{ "status": "approved", "partnerId": "ptn_001" }` | `200 OK` | `422 Unprocessable Entity` for invalid transition |
| Lending Partners | List partners | GET | `/api/v1/lending-partners` | Query: `?active=true&currency=RWF` | `200 OK` | `400 Bad Request` |
| Disbursements | Create disbursement | POST | `/api/v1/disbursements` | `{ "financingRequestId": "fin_001", "partnerId": "ptn_001", "amount": 500000, "currency": "RWF" }` | `201 Created` | `422 Unprocessable Entity` if request not approved |
| Disbursements | Update status | PATCH | `/api/v1/disbursements/{disbursementId}` | `{ "status": "completed", "paymentReference": "MOMO-001" }` | `200 OK` | `422 Unprocessable Entity` for invalid transition |
| Remittances | Create transaction | POST | `/api/v1/remittance-transactions` | `{ "merchantId": "mch_001", "direction": "outbound", "counterpartyName": "David K.", "amount": 120000, "currency": "RWF", "destinationCountry": "Kenya" }` | `201 Created` | `422 Unprocessable Entity` if route unsupported |
| Remittances | Get transaction | GET | `/api/v1/remittance-transactions/{transactionId}` | None | `200 OK` | `404 Not Found` |

## Validation and Error Strategy

The API should return consistent JSON errors so client applications and partner systems can understand what went wrong.

Standard error format:

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

Important checks:

| Check | Status Code | Example |
|---|---:|---|
| Missing required field | `400 Bad Request` | Merchant request without `businessName` |
| Duplicate phone number | `409 Conflict` | Two merchants using the same phone number |
| Invalid status value | `400 Bad Request` | Status sent as `paid_out` |
| Business rule violation | `422 Unprocessable Entity` | Suspended merchant requests financing |
| Missing resource | `404 Not Found` | Unknown merchant ID |
| Invalid status transition | `422 Unprocessable Entity` | Rejected request changed directly to disbursed |

## Trade-off Rationale

### Separating Financing Requests and Disbursements

I chose to keep `FinancingRequest` and `Disbursement` as separate resources. Approval and money movement are related, but they are not the same event. A request can be approved before funds are actually sent, and a payment can fail or be reversed.

The benefit is better tracking and a clearer audit trail. The trade-off is that developers may need to fetch more than one resource to see the full financing lifecycle. This is reasonable because financial systems need careful tracking more than they need everything stored in one object.

### Using PATCH for Status Changes

I chose `PATCH` for updates such as merchant verification, financing review, and payment status changes. This allows clients to update only the fields that changed.

The trade-off is that status transitions need careful validation. The API must prevent impossible workflows, such as moving a rejected request directly to disbursed. This is still reasonable because controlled status changes make the API safer and easier to test.


### Keeping Remittance Transactions Separate

I chose to keep remittance transactions separate from merchant profiles.

A merchant profile stores information about the business, like name, phone number, country, and verification status. A remittance transaction stores money movement, like amount, currency, receiver, and payment status.

The good side is that the merchant record stays clean and easy to understand. Each money transfer can be tracked on its own.

The trade-off is that if someone wants to see both the merchant details and all their transactions, the system may need to look in more than one place.


## Final Reflection

<!-- Write this section yourself before submitting.

Possible points to focus on:
- Which resource was hardest for you to design and why?
- Why does a financial API need strict validation?
- Why did you separate credit requests, disbursements, and remittance transactions?
- What would you improve if this design became a real implementation next week?
-->

## AI Use Appendix

AI tool used: GPT-5.5 through ChatGPT/Codex.

What I asked it to help with: I used GPT-5.5 to brainstorm and better understand the assignment goals, especially how the Imara business context could be translated into REST resources, endpoints, validation rules, and trade-off decisions.

Ideas I kept: I kept the idea of organizing the design around merchants, field agents, financing requests, lending partners, disbursements, and remittance transactions. I also kept the idea of using consistent error responses.

Ideas I changed or rejected: I shortened the design, selected the core resources I thought fit the first version of the platform, and kept the document focused on a blueprint instead of implementation code.

Why the final design reflects my own judgment: GPT-5.5 helped me brainstorm and clarify the project requirements, but I reviewed the assignment brief and made the final choices about the resources, business rules, endpoint examples, and trade-offs included in this document.
