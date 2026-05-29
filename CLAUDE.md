---
name: xero
description: >
  Read accounting data (invoices, contacts, bank transactions, journals, etc.)
  from the Xero cloud accounting platform via the Xero Accounting API.
type: api
---

# Xero

Cloud accounting platform. This connector reads accounting data via the Xero Accounting API (`https://api.xero.com/api.xro/2.0/`).

## Authentication

### OAuth2 Authorization Code
- Client app required: yes (register an app in the Xero Developer portal; provides Client ID + Client Secret)
- Authorize URL: `https://login.xero.com/identity/connect/authorize`
- Token URL: `https://identity.xero.com/connect/token` (HTTP Basic auth of client_id:client_secret)
- Refresh: supported via the `offline_access` scope
- Header format: `Authorization: Bearer ${access_token}`
- Scopes requested: `openid profile email offline_access accounting.transactions.read accounting.contacts.read accounting.settings.read accounting.journals.read accounting.reports.read`

## Post-Auth Steps

Required. After token exchange, call `GET https://api.xero.com/connections` to list authorised organisations (tenants). The user selects one; its `tenantId` is stored as `connection.selections.tenantId`. Every Accounting API request must send the `Xero-tenant-id: ${connection.selections.tenantId}` header. The `/connections` call itself must NOT send the tenant header.

## Available Endpoints

All read-only. `page` = page-number pagination (100/page, max pageSize 1000); `keyset` = advance by JournalNumber; `none` = single call. Incremental endpoints filter on the `If-Modified-Since` header.

| Endpoint | Method | Pagination | Incremental cursor | Description |
|----------|--------|------------|--------------------|-------------|
| accounts | GET | none | UpdatedDateUTC | Chart of accounts |
| bank_transactions | GET | page | UpdatedDateUTC | Spend/receive money bank transactions |
| contacts | GET | page | UpdatedDateUTC | Customers and suppliers |
| credit_notes | GET | page | UpdatedDateUTC | Credit notes |
| invoices | GET | page | UpdatedDateUTC | Sales & purchase invoices |
| items | GET | none | UpdatedDateUTC | Products/services (items list) |
| journals | GET | keyset (offset by JournalNumber) | CreatedDateUTC | General-ledger journals |
| manual_journals | GET | page | UpdatedDateUTC | Manual journals |
| organisations | GET | none | — | Organisation settings/metadata |
| payments | GET | page | UpdatedDateUTC | Payments |

## Rate Limits

- 60 requests per 60 seconds per organisation (tenant). Xero also enforces 5,000 calls/day per tenant and a concurrency cap.

## Caveats

- Responses wrap records in a root object (e.g. `{"Invoices": [...]}`) alongside `Id`/`Status`/`ProviderName`/`DateTimeUTC` metadata.
- Timestamps use Xero's `/Date(…)/` format; date fields are surfaced as strings.
- Nested objects/arrays (line items, contacts, addresses, etc.) are surfaced as JSON.
- Each connection targets a single tenant; connect once per organisation.
