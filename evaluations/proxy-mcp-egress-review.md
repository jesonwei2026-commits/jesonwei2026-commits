# Proxy-pool MCP egress-path review (Oct 2026)

Subject: [99apps-id/proxy-mcp](https://github.com/99apps-id/proxy-mcp) — a stdio MCP server that routes outbound requests through a configured proxy pool.

Method: read the README in full and every source file line by line (main.go, mcp.go, pool.go, proxy.go, pool_test.go, example config, go.mod, LICENSE). No install, no run, no credentials, no production config touched. README claims are kept separate from code evidence throughout.

## What it does
- Single argument `-config <proxies.json>`; serves three tools over stdio JSON-RPC: `proxy_request` (fetch a URL through one pool proxy), `proxy_health` and `proxy_list` (per-proxy alive/latency/failures/successes/success rate).
- Proxy selection is greedy best-score, not round-robin: score = Successes x 2 - Failures, lifetime cumulative with no decay, so the pool converges on one historically best proxy instead of rotating.
- Health checks every 30s against a fixed probe URL; any status below 500 counts as alive, so 403/429/404 all count as alive. Probe success does not raise the success counter; only real `proxy_request` calls do.
- Response bodies are capped (200KB read, 20KB preview returned); no streaming, no binary, no cookie persistence; default User-Agent is a fixed tool string.

## Findings (code-verified)
1. SOCKS5 entries do not work: the code dials the `socks5://host:port` string directly over TCP with no SOCKS5 handshake and no credential handling, although the dependency is declared.
2. Success means transport success only (`err == nil`): HTTP 403, 429 and 500 responses all raise the success counter, so a persistently blocked proxy keeps being preferred.
3. One call pins one proxy: no in-call retry or switch; "failover" only happens on the next call.
4. Fail-open: when no proxy is alive, the server returns a pseudo entry `direct` and the request goes out from the local machine with no error and no interception; the bypass is invisible in statistics.
5. TLS verification is globally disabled (`InsecureSkipVerify: true`) for all HTTPS requests made through it.
6. Missing `method` crashes the server: the parameter is declared optional (default GET) but read with an unguarded type assertion; a nil value panics and the serve loop has no recover.
7. Declared config fields `weight`, `concurrency`, `username`/`password` and `country` are never used by selection or limiting code; country-based or target-based selection does not exist.
8. SSRF protections claimed in the README (private-IP and metadata-IP filtering) do not exist anywhere in the code; any absolute http(s) URL can be requested, with redirects followed by default.

## Verdict
Useful as a control-plane design reference (config shape, health-check and scoring ideas). Not a clean-egress solution: rotation only chooses among the IPs already in the pool and creates no clean IP, so a pool of flagged datacenter IPs stays flagged no matter how it rotates. Any future use would need, at minimum: status-code-aware scoring, working SOCKS5 or HTTP-only mode, TLS verification restored, a default for `method`, fail-closed behavior, and an external target allowlist.

## License note
README states Apache-2.0; the LICENSE file is indeed the full Apache-2.0 text (copyright 2024 99Apps), while GitHub's license detection field shows NOASSERTION (a detection miss, not a different license). The repo has 0 releases, so no prebuilt binary is available; it must be built from source.
