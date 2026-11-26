# Secure REST API Integration for ChatGPT Plugins

## Purpose

Build MCP-backed ChatGPT plugins that call external REST APIs while keeping credentials outside the plugin package, source repository, prompts, and tool definitions.

## Core architecture

```text
ChatGPT
  |
  v
MCP server (your backend)
  |  reads secret from runtime secret store
  |  constructs HTTPS request
  v
External REST API
  |
  v
MCP tool result
  |
  v
ChatGPT
```

The MCP server is the trusted boundary. Do not let model-generated arguments choose arbitrary secret names, environment-variable names, or unrestricted destination URLs.

## Secret management rules

1. Never hardcode API keys, passwords, private tokens, or signing secrets in source code, manifests, SKILL.md files, prompts, tool schemas, logs, or plugin archives.
2. Do not commit a real `.env` file. Commit only a placeholder `.env.example` such as `EXTERNAL_API_KEY=`.
3. For local development, load secrets from an ignored `.env` file or the operating system's secret store. Add `.env` and other credential files to `.gitignore`.
4. In production, use the hosting provider's encrypted environment-variable facility or a dedicated secret manager (for example, Vercel Environment Variables, AWS Secrets Manager, or an equivalent vault).
5. Read secrets at runtime on the server. Never return secret values to ChatGPT, end users, client-side code, tool responses, or logs.
6. Do not expose a generic tool that reads arbitrary environment variables. Bind each integration to a specific, server-selected secret.
7. Rotate credentials when exposure is suspected. Revoke old credentials and review repository history and deployment logs.
8. Validate and authorize user-specific credentials separately. Encrypt them at rest, scope access to the authenticated user, and never share one user's secret with another.

## REST request implementation

Use a normal HTTP client (for example, Node.js `fetch`) from the server. Executing the `curl` binary is usually unnecessary; it provides no security advantage over an HTTP client.

Example:

```javascript
const API_BASE_URL = "https://api.example.com";

export async function fetchAccountData() {
  const apiKey = process.env.EXTERNAL_API_KEY;

  if (!apiKey) {
    throw new Error("External API integration is not configured");
  }

  const response = await fetch(new URL("/v1/account", API_BASE_URL), {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
    signal: AbortSignal.timeout(10_000),
  });

  if (!response.ok) {
    // Do not include authorization headers or secret values in errors.
    throw new Error(`External API request failed: HTTP ${response.status}`);
  }

  return response.json();
}
```

For real integrations, also validate response schemas, handle rate limits and retries carefully, cap response sizes, and avoid returning unnecessary personal or sensitive data.

## Prevent SSRF and arbitrary proxying

If a tool accepts a URL or host from the model/user:

- Prefer fixed, allowlisted API hosts and fixed endpoint paths.
- Reject localhost, private, link-local, and metadata-service IP ranges.
- Validate DNS resolution and every redirect; ideally disable redirects for credentialed requests.
- Do not forward authorization headers to a different origin.
- Restrict methods, paths, content types, request sizes, and timeouts.
- Never build an unrestricted "fetch any URL with my API key" tool.

## MCP tool design

Expose narrow, task-oriented tools such as `get_account`, `list_items`, or `create_ticket`, not a universal shell or unrestricted HTTP proxy.

Tool arguments should describe the operation and user-supplied data only. The server determines:
- Which API host and endpoint to call.
- Which HTTP method is allowed.
- Which credential to use.
- Which headers are required.
- Which response fields may be returned.

Return structured, minimal results. Sanitize errors and never echo request headers or credentials.

## Developer-owned vs user-owned API keys

### Developer-owned key

The plugin operator configures one secret in the backend hosting environment. All authorized calls use that server-side credential. Apply quotas, rate limits, abuse monitoring, and per-user authorization because usage may be billed to the operator.

### User-owned key

Each user supplies their own credential through a secure connection or account-configuration flow. Store it using a protected credential store, associate it with the authenticated user, and never put it in conversation text or ordinary tool arguments unless the platform's documented authentication flow explicitly requires it.

Support both only when there is a clear product requirement. Prefer OAuth where the upstream API supports it.

## Deployment checklist

- [ ] No real secrets in Git, plugin ZIPs, manifests, prompts, or client bundles.
- [ ] `.env` and credential files are ignored by Git.
- [ ] Production secrets are configured in the hosting platform's secret store.
- [ ] Missing secrets fail closed with a sanitized error.
- [ ] HTTPS is enforced.
- [ ] API hosts and endpoints are allowlisted.
- [ ] Redirects, DNS, and SSRF risks are handled.
- [ ] Authentication and authorization are checked for every user operation.
- [ ] Rate limits, timeouts, payload limits, and response validation are implemented.
- [ ] Logs redact authorization headers, tokens, and sensitive request bodies.
- [ ] Secret rotation and incident-response procedures exist.

## Key principle

The plugin package describes capabilities; the backend holds authority. Keep secrets server-side, constrain what tools can do, and expose only the minimum data required for the task.
