# Backlog and Known Issues

Moved from CLAUDE.md, 2026-09-08, to keep the instruction core short. Covers open tech debt and unbuilt tool ideas; shipped work is in CHANGELOG.md.

## Known Issues / Tech Debt

- **User identity retrieval:** The OIDC `/identity/connect/userinfo` endpoint (with `openid profile email` scopes) is the primary method. Falls back to `/entity/auth/25.200.001/UserSecurityInfo` which may not exist on all instances. If both fail, username defaults to a UUID-based key (breaks token reuse across sessions).
- **Acumatica system entities not available via contract API:** `User`, `UserRole`, and screen-based API (`/entity/Default/.../screen/SM201010`) all return 404 on SaaS instances. The canary GI approach for the access gate was adopted because of this limitation (there's no API to query role membership).
- **`$select` on some entities causes Acumatica 500:** Some entities (e.g., Payment) return internal server errors when `$select` is used with certain field names. The `acumatica_list_entities` tool auto-retries without `$select` when this occurs.
- **`substringof(...) eq true` silently returns `[]`:** Acumatica's contract-REST `$filter` parser returns an empty set (HTTP 200, no error) for a boolean string function compared to a literal — `substringof('X', F) eq true` / `startswith(...) eq true` / `endswith(...) eq true` — but the *bare* function works. Models habitually append `eq true` (valid OData v3). `normalizeODataFilter()` (`src/lib/odata-filter.ts`) strips it server-side for both `acumatica_list_entities` and `acumatica_run_inquiry`. `eq false` is left verbatim — the only equivalent negation (`not substringof(...)`) 500s on the contract API. NOT a transport/encoding bug (an early parens-encoding hypothesis was disproven live).
- **Complex document entities can't be server-side `$filtered` on non-key fields:** PurchaseOrder, Shipment, PhysicalInventoryCount (and any filter that reaches a child collection, e.g. `StockItem/CrossReferences/AlternateID`) fail in two ways — (A) HTTP 500 from the OData filter binder (`CannotOptimizeException`, "type conversions not supported", "not a single value", "key not present") or (B) a *silent* `[]` even when matching rows exist (e.g. `substringof` on PurchaseOrder `VendorID`). A keyed filter (`OrderNbr`/`ShipmentNbr eq '...'`, topN=1) is optimizable and works; broad search must use a Generic Inquiry. `getFilterErrorKind()` (`src/lib/complex-entities.ts`) classifies mode-A 500s into a structured `filterNotApplicable` error; mode-B empties on the known-list entities get a `possibleFalseNegative` warning so the model doesn't conclude "no such record exists." The known-list is hardcoded — see `docs/upgrading-acumatica.md` §7.
- Old Entra ID secrets may still exist on Cloudflare — clean up with `wrangler secret delete ENTRA_CLIENT_ID`, etc.
- **Zod schema constraint:** MCP tool parameter schemas MUST use only simple types (`z.string()`, `z.string().optional()`, `z.string().default("value")`). Complex types like `z.record()`, `z.unknown()`, `z.number()` cause MCP SDK JSON Schema serialization failures and tools won't appear in client discovery. Use `z.string()` with manual `parseInt()` in the handler for numeric parameters.
- **ChatGPT CIMD bug (as of April 2026):** ChatGPT's MCP client sees `client_id_metadata_document_supported: true` in our metadata but fails to complete CIMD (it doesn't have its own metadata document URL) and does not auto-fallback to DCR. Users must manually select DCR when adding the server in ChatGPT. Our server correctly advertises both — this is a ChatGPT client-side issue.
- **Claude.ai tool list caching:** Claude.ai may cache the tool list from a previous Durable Object session. If tools appear stale, disconnect and reconnect the MCP server in Claude.ai to force a fresh `init()` call.
- **Claude.ai re-auth is not silent, and can get stuck:** When the server revokes a grant (dead Acumatica token → `ReauthRequiredError`), the 401 + `WWW-Authenticate` *should* let the client re-run OAuth invisibly. In practice Claude.ai surfaces a **reconnect prompt** rather than re-authing silently, and after a few failed attempts it caches the connector in a dead state and stops prompting entirely. Recovery for an **org-managed** connector is a **personal disconnect → reconnect** (the org-level "delete" is not available/needed to individual users) — this clears the stuck personal grant and starts a fresh `/authorize` flow. The `0.33.0` TokenManager DO removes the *spurious* revokes (rotation races) that were triggering this; genuine dead-token revokes still prompt.
- **`/authorize` error mapping on a bad/unfetchable `client_id` (fixed 0.38.5):** `app.get("/authorize")` wraps `parseAuthRequest()` in try/catch. A CIMD-fetch failure → **502** ("client's metadata endpoint down, not this server"); a malformed/invalid `client_id` → **400**; both log `authorize_parse_failed` (client_id + error, no secrets) for `wrangler tail`. Previously the throw surfaced as an opaque HTTP 500. This is **not** harmless: when Claude.ai's CIMD metadata endpoints (`claude.ai/oauth/mcp-oauth-client-metadata`, `…/claude-code-client-metadata`) had a transient 503 outage (observed 2026-07-06), the server-side CIMD fetch failed and *every* CIMD client (Claude.ai web + current Claude Desktop) hit the 500 and couldn't connect — DCR clients were unaffected. The 502 now makes that failure mode self-diagnosable from the server's own response/logs.
- **No description metadata for Generic Inquiries.** *(Addressed in 0.37.0 via the GI registry — see "GI Tool Gating & Registry". The curation now lives in the `MCPGIs`/`MCPGIFields` feed GIs (`UsrAIDescription` fields), surfaced through the lazy registry; this is a hybrid of cures #2 and #3 below. The note is kept for the underlying-platform context.)* The Acumatica GI Design form (SM208000) has no free-text "Description" field on the header — only `Inquiry Title` (a short label, often just a prettified name) and `Site Map Title` (set only for nav-pinned GIs). The OData GI service document returns `{name, url}` only; nothing richer is exposed. As a result, `acumatica_list_generic_inquiries` surfaces GIs by name alone, leaving the model to guess which GI matches a user's intent. Parametrized GIs are already excluded at list time (see `generic-inquiry-discovery.ts` — `$metadata` is scanned for `FunctionImport Name="..._WithParameters"` entries and those are filtered out), so the gap is narrowly about selection context for the surviving non-parametrized GIs. Potential cures:
  1. **MCP-side curation map.** KV-backed `gi_descriptions:{name} → text` edited from the admin console. Filter the list to GIs that have a description, inject the description into the response. Curation lives where it's consumed, zero Acumatica-side change. Downside: descriptions invisible inside Acumatica; admins must maintain a second list.
  2. **Acumatica-side meta-GI.** Admin publishes a `MCPGIIndex` GI whose rows are `(Name, Description)` pulled from a custom table or hand-maintained dataset. Visible inside Acumatica, but heavier setup and the descriptions live separately from the GI definitions themselves.
  3. **Extract GI definition (XML / GIQL) via API and auto-generate descriptions.** If Acumatica exposes the GI design body — tables joined, filters, output columns — through a screen-based API, OData $metadata annotations, or an export endpoint, feed each GI's structure to Claude Code (or any model) and have it produce a one-line description plus parameter/usage notes from the query itself. Cache the generated text alongside the existing GI metadata cache. Most automated of the three; requires verifying which API surface (if any) returns the design body on SaaS — historically system entities like `GenericInquiry` / `GIDesign` are not in the contract API (see the "system entities" note above), so this path likely depends on either a non-public endpoint or an admin-published export.

## TODO — Remaining Project Work

### High Priority — Features
- [~] Add write tools: Create/update Sales Orders, Customers, Vendors (per project brief Phase 2) — write-tool infrastructure (`WRITER_TOOLS` registry + `runWriter`, kill-switch, dry-run gate, top-level & nested allowlist, R2-persisted mutation audit) + first tool `acumatica_create_or_update_customer` landed 0.40.0; Vendor / SalesOrder are one registry entry each
- [ ] Add action tools: Release Invoice, Confirm Shipment (per project brief Phase 3)
- [x] Transparent re-auth when refresh token expires — `ReauthRequiredError` revokes the MCP grant so the client silently re-runs OAuth instead of a manual disconnect/reconnect (0.32.0)

### Low Priority — Read-Only Tools

**Financial (additional):**
- [ ] AccountSummaryInquiry — GL account balances by period/ledger
- [ ] AccountDetailsForPeriodInquiry — GL transaction detail for a period
- [ ] CashSale — point-of-sale cash transactions
- [ ] CashTransaction — bank deposits, withdrawals, transfers
- [ ] Budget — GL budget lines by period
- [ ] Ledger — ledger master data (actual, budget, statistical)
- [ ] Subaccount — sub-account segments
- [ ] Tax — tax ID master data
- [ ] TaxCategory — tax category definitions
- [ ] TaxZone — tax zone definitions

**Sales (additional):**
- [ ] CustomerLocation — customer ship-to/bill-to locations
- [ ] CustomerClass — customer classification defaults
- [ ] CustomerPaymentMethod — stored payment methods
- [ ] SalesPricesInquiry — item price lookup
- [ ] Discount / DiscountCode — discount rules

**Purchasing (additional):**
- [ ] VendorClass — vendor classification defaults
- [ ] VendorPricesInquiry — vendor price lookup

**Inventory (additional):**
- [ ] InventoryAllocationInquiry — allocation breakdown (on hand, available, on PO, etc.)
- [ ] StorageDetailsInquiry / StorageDetailsByLocationInquiry — lot/serial detail
- [ ] ItemWarehouse — per-warehouse item settings
- [ ] KitSpecification — kit/BOM definitions
- [ ] TransferOrder — inter-warehouse transfers
- [ ] InventoryAdjustment / InventoryIssue / InventoryReceipt — inventory transactions

**Other:**
- [ ] FinancialPeriod / FinancialYear — fiscal calendar
- [ ] Currency — currency master data
- [ ] ShipVia / ShippingTerm / ShippingZones — shipping config

### Low Priority — Infrastructure
- [ ] Add Attachment upload/download tools
- [ ] Remove old Entra ID secrets from Cloudflare (`wrangler secret delete`)
- [~] Add unit tests — `test/` harness added (`npm test`, node --test); covers filter normalization + complex-entity detection. Broader coverage still pending.
- [ ] Add CI/CD pipeline

