# Decisions

Architectural decisions and their reasoning, moved out of CLAUDE.md to keep the instruction core short. New entries can be appended the same way: a heading naming the decision, then the reasoning.

## Historical Note: Why We Removed Microsoft Entra ID

(Moved from CLAUDE.md, 2026-09-08.)

The initial design used a two-login chained OAuth flow: users first authenticated via Microsoft Entra ID (to identify who they are), then were chained to Acumatica OAuth (to get API permissions). This required a separate Entra app registration, three callback routes, and intermediate state management in KV.

We removed Entra ID entirely because:
- **It was redundant.** Since every user must authenticate with Acumatica anyway (to get a per-user API token with their role-based permissions), the Entra login added no value — Acumatica already knows who the user is.
- **Acumatica can use Entra SSO natively.** If an Acumatica instance is configured with Entra SSO, users still get the Microsoft login experience — it just happens through Acumatica's own login page, not through our MCP server.
- **Simpler flow.** One login instead of two. One callback route instead of three. No Entra secrets to manage.

Old Entra-related secrets (`ENTRA_CLIENT_ID`, `ENTRA_CLIENT_SECRET`, `ENTRA_TENANT_ID`) may still exist on the Cloudflare side and should be cleaned up with `wrangler secret delete`.
