# Hi there 👋

Building a long-term free LLM gateway and tooling. Tracking open-source AI, self-hosting and free infrastructure.
## Featured work

### Proxy-pool MCP egress-path review (Oct 2026)
Code-level review of [99apps-id/proxy-mcp](https://github.com/99apps-id/proxy-mcp), a stdio MCP server that routes outbound requests through a proxy pool. Read the README and every source file line by line; no install, no run, no credentials involved.

**Verdict:** useful as a control-plane design reference, but not a clean-egress solution — rotation only chooses among the IPs already in the pool; it creates no clean IP.

**Code-level findings:**
- SOCKS5 entries are non-functional (no SOCKS5 handshake in code).
- HTTP 403 / 429 / 500 responses count as success, so a persistently blocked proxy keeps its score and keeps getting picked.
- No in-call retry or proxy switch; when all proxies are dead it fails open to a direct connection.
- TLS certificate verification is globally disabled.
- Omitting the optional `method` parameter crashes the server (uncaught type assertion).
- The README's SSRF protections (private/metadata IP filtering) do not exist anywhere in the code.
- LICENSE file is Apache-2.0 text, though GitHub's detection field shows NOASSERTION; the repo has 0 releases, so no prebuilt binary exists.

Full note: [evaluations/proxy-mcp-egress-review.md](evaluations/proxy-mcp-egress-review.md)
