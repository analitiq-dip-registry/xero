---
name: xero
description: >
  Read and write accounting data (invoices, contacts, bank transactions, journals, etc.)
  from the Xero cloud accounting platform via the Xero Accounting API.
type: api
---

# Xero

Cloud accounting platform. This connector reads and writes accounting data via the Xero Accounting API (`https://api.xero.com/api.xro/2.0/`).

## Authentication

### OAuth2 Authorization Code
- Client app required: yes (platform-managed; `client_id` and `client_secret` are provisioned by the Analitiq platform — users do not supply them)
- Authorize URL: `https://login.xero.com/identity/connect/authorize`
- Token URL: `https://identity.xero.com/connect/token` (HTTP Basic auth of client_id:client_secret)
- Refresh: supported via the `offline_access` scope
- Header format: `Authorization: Bearer ${access_token}`
- Scopes requested:
  `openid profile email offline_access accounting.settings accounting.contacts accounting.invoices accounting.banktransactions accounting.payments accounting.manualjournals accounting.journals.read`

**Granular scopes.** These are Xero's granular scopes, not the older broad ones. The deprecated
`accounting.transactions[.read]` and `accounting.reports.read` are no longer requested. A scope
without a `.read` suffix grants read **and** write; `accounting.journals.read` stays read-only
because the journals resource has no write path.

Coverage worth knowing, since it is not guessable from the scope names:
`accounting.invoices` covers **CreditNotes** and **Items** as well as Invoices;
`accounting.settings` covers **Accounts** and **Organisation**.

Two operational consequences:
- Scopes are **additive and irreducible** — an existing connection cannot have scopes narrowed
  without revoking the token and re-consenting.
- Existing connections authorised under the old broad scopes keep working, but must re-consent
  to pick up the granular set. Xero retires the broad scopes in **September 2027**.

## Post-Auth Steps

Required. After token exchange, call `GET https://api.xero.com/connections` to list authorised organisations (tenants). The user selects one; its `tenantId` is stored as `connection.selections.tenantId`. Every Accounting API request must send the `Xero-tenant-id: ${connection.selections.tenantId}` header. The `/connections` call itself must NOT send the tenant header.

## Available Endpoints

`page` = page-number pagination (`page`/`pageSize`, sibling `pagination` envelope); `keyset` =
advance by JournalNumber; `none` = single call. Incremental endpoints filter on the
**`If-Modified-Since` request header** (not a query param), format `2020-02-06T12:17:43.202-08:00`.

| Endpoint | Pagination | Incremental cursor | Write modes | Description |
|----------|------------|--------------------|-------------|-------------|
| accounts | none | UpdatedDateUTC | insert | Chart of accounts |
| banktransactions | page | UpdatedDateUTC | insert, upsert | Spend/receive money bank transactions |
| contacts | page | UpdatedDateUTC | insert, upsert | Customers and suppliers |
| creditnotes | page | UpdatedDateUTC | insert, upsert | Credit notes |
| invoices | page | UpdatedDateUTC | insert, upsert | Sales & purchase invoices |
| items | none | UpdatedDateUTC | insert, upsert | Products/services (items list) |
| journals | keyset (offset by JournalNumber) | CreatedDateUTC | — | General-ledger journals |
| manualjournals | page | UpdatedDateUTC | insert, upsert | Manual journals |
| organisation | none | — | — | Organisation settings/metadata |
| payments | page | UpdatedDateUTC | insert | Payments |

**No maximum `pageSize`.** Xero documents a *default* of 100 but states no maximum anywhere in
first-party sources, so no pagination `limit.max` is declared. (Earlier revisions of this file
claimed a max of 1000; that was ungrounded — do not reintroduce it.)

### Two resources are insert-only, by provider constraint

- **accounts** — Xero has no collection-level `POST /Accounts`. The item-scoped
  `POST /Accounts/{AccountID}` is structurally inexpressible: `path_params` accepts only
  `{from_param}` bindings, so a per-record `AccountID` cannot reach the path.
- **payments** — Xero documents **no update operation** for an existing payment. The
  collection `POST /Payments` is `createPayment` (a create, not update-or-create), and the
  item-level `POST /Payments/{PaymentID}` is `deletePayment`.

For the other six writable resources, `PUT /<Resource>` creates and `POST /<Resource>` is a
genuine update-or-create matched on the resource GUID.

### Write shape

Writes send a **single-record envelope** — `{"Invoices": [{...}]}` — rather than batching.
This is deliberate: `batching` and `idempotency` are mutually exclusive under the contract, and
Xero's documented `Idempotency-Key` header is worth more than throughput here, especially since
no first-party Xero batch cap is documented to size a batch against.

`unitdp=4` is sent where Xero supports it (**banktransactions, creditnotes, invoices, items**)
to preserve four-decimal unit amounts inside line items. It does **not** apply to
manualjournals, payments, contacts, accounts, journals or organisation.

## Type mapping

- **Timestamps are properly typed**, not strings. Xero runs four distinct temporal shapes and
  each maps to its own canonical type:
  | Wire shape | Example | Canonical |
  |---|---|---|
  | `<X>DateUTC` MS-JSON with time | `/Date(1573755038314)/` | `Timestamp(MILLISECOND, UTC)` |
  | `<X>DateUTCString` ISO-8601 with `Z` | `2019-11-14T18:10:38Z` | `Timestamp(MICROSECOND, UTC)` |
  | Business dates, MS-JSON date-only | `/Date(1481846400000+0000)/` | `Date32` |
  | `DateString` ISO-8601 zoneless | `2016-12-16T00:00:00` | `Timestamp(MICROSECOND)` |
- **Money is `Decimal128(19, 4)`.** Scale 4 follows Xero's own `unitdp` domain (2 and 4).
- **Rates and quantities stay `Float64`** — `CurrencyRate`, `CISRate`, `Discount`,
  `QuantityOnHand`. Forcing an FX rate to 4dp would silently round e.g. `0.68123456` to
  `0.6812`, which is strictly worse than a float.
- Nested objects/arrays (line items, addresses, balances, tracking) are opaque `Json` and carry
  no per-field types — so money *inside* them is not individually typed.

## Rate Limits

- 60 requests per 60 seconds per organisation (tenant), 5,000 calls/day per tenant, and a cap of
  5 concurrent requests (declared as the connector's `concurrency.max_connections`).

## Caveats

- Responses wrap records in a root object (e.g. `{"Invoices": [...]}`) alongside
  `Id`/`Status`/`ProviderName`/`DateTimeUTC` metadata.
- Redundant `<X>String` date siblings are dropped where the MS-JSON field already carries the
  value at its true grain — one column per value, not two.
- Write `success_when` checks the envelope `Status == "OK"`, which detects **call-level**
  failure. With Xero's default `summarizeErrors=false`, per-record validation failures still
  return HTTP 200 with `Status: "OK"` and populate per-record `ValidationErrors` /
  `StatusAttributeString`. `StatusAttributeString` **is** surfaced on read, so a rejected
  record is still detectable there; `ValidationErrors` and `Warnings` are **not** modelled —
  they are write-response artifacts, so the specific reason for a rejection is not landed.
- **Columns removed relative to v0.1.0.** `ValidationErrors` and `Warnings` were dropped from
  every endpoint that declared them (write-response artifacts, not read columns), and
  `Attachments` was dropped from `invoices` and `manualjournals` for cross-resource parity —
  attachments are served by the item-level `/{id}/Attachments` path family, which this
  connector does not model. Landed tables lose those columns on upgrade.
- `organisation` deliberately omits Xero's `APIKey` property. It is a live per-organisation
  credential with no analytical value, and the contract has no field-level masking affordance —
  authoring it would replicate a working credential into the destination warehouse in plaintext.
- `contacts` does not send `includeArchived`, so archived contacts are not extracted. Xero
  documents no default for the param, and it was left ungrounded rather than guessed.
- `invoices` returns `LineItems` as an empty array on the **list** endpoint; line items populate
  only on the single-invoice GET, which this connector does not model.
- Each connection targets a single tenant; connect once per organisation.
