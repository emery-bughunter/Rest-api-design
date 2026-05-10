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