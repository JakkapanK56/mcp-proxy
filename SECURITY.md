# Security deployment guide (JakkapanK56 fork)

## Secure-by-default network policy

Non-loopback HTTP listeners require an inbound Bearer token **before any local MCP stdio process starts**.
Local-only binds on `127.0.0.1`, `::1`, and `localhost` retain the upstream unauthenticated behavior.

Generate a high-entropy token with `python -c 'import secrets; print(secrets.token_urlsafe(48))'`.
Store the token in a secret manager, then set `MCP_PROXY_AUTH_TOKEN` in the proxy environment.
Do not pass bearer secrets on the command line or commit them to configuration files.

Example after setting `MCP_PROXY_AUTH_TOKEN` securely:

```bash
mcp-proxy --host=0.0.0.0 --port=8080 \
  --allowed-host mcp.example.com:8080 \
  --allow-tool read_file --allow-tool list_files \
  -- uvx your-reviewed-mcp-server
```

Use a TLS-terminating reverse proxy or an authenticated private tunnel for all non-local traffic.
The shared Bearer token is not OAuth, per-user identity, RBAC, or per-agent authorization.

## Defense in depth and limitations

- Repeat `--allow-tool NAME` to filter `tools/list` and reject disallowed `tools/call` requests.
  Tool names are matched exactly; the allowlist does **not** restrict resources, prompts, or other MCP methods.
- Upstream tool errors are redacted by default, because they may contain secrets or local paths.
  `--expose-tool-errors` is an explicit unsafe compatibility override.
- `--allow-unauthenticated-remote` overrides the remote-bind guard. Only use it in isolated test networks.
- DNS rebinding protection, Host header validation and CORS are **not** authentication.
  Avoid broad `--allow-origin` policies on sensitive deployments.
- Avoid `--pass-environment`; it may send sensitive parent environment values to local MCP subprocesses.
- Review and pin all `uvx`/`npx` packages, restrict filesystem mounts, and run as a non-admin OS user.
- Never expose privileged shell or filesystem tools based on the shared token alone.
- TLS, per-client authorization, rate limits, OAuth/OIDC, mTLS, audit retention, and process sandboxing
  are not implemented by this fork and need to be provided by the deployment environment.

## Compatibility and verification

Previously unauthenticated non-loopback deployments must now configure a token or explicitly opt out.
Tool error text that was previously forwarded is now replaced with a generic error by default.
Startup logs no longer show child command-line arguments.

```bash
uv run --frozen pytest -q tests/test_transport_security.py tests/test_proxy_server.py tests/test_cli_arguments.py
```

Check that remote binds without a token fail before child startup, unauthenticated HTTP requests
receive 401 when auth is enabled, and denied tools cannot be invoked even if a client supplies the name.
