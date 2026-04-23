# MCP OAuth — Auth0

<p>
  <a href="https://github.com/mcp-use/mcp-use">Built with <b>mcp-use</b></a>
  &nbsp;
  <a href="https://github.com/mcp-use/mcp-use">
    <img src="https://img.shields.io/github/stars/mcp-use/mcp-use?style=social" alt="mcp-use stars">
  </a>
</p>

A production-ready MCP server template authenticating users via Auth0 OAuth 2.1. Implements bearer token authentication with JWKS-based JWT verification and scope-based authorization (RBAC).

## Features

- **OAuth 2.1 with PKCE** — Authorization Code Flow with PKCE
- **JWT verification** — JWKS-based token verification using Auth0's public keys
- **Bearer token authentication** — Verified Auth0 access tokens on every MCP request
- **Role-based access control** — Scope-based permissions for granular tool access
- **User context in tools** — `ctx.auth.user` populated from JWT claims

## Prerequisites

> **Note**: Auth0's MCP features are currently in Early Access. To join the program, complete [this form](https://forms.gle/hvJ1ZRLmHr9YjV2a9).

1. **Auth0 account** — Sign up at [auth0.com](https://auth0.com) (free tier available)
2. **Auth0 CLI** — Install the [Auth0 CLI](https://auth0.github.io/auth0-cli/) for configuration
3. **Node.js 20+** (22 recommended)
4. **pnpm 10+**
5. **jq** — JSON processor for the CLI scripts below ([install](https://jqlang.org/download/))

## Setup

### 1. Configure your Auth0 tenant

Log in to the Auth0 CLI with the required scopes:

```bash
auth0 login --scopes "read:client_grants,create:client_grants,delete:client_grants,read:clients,create:clients,update:clients,read:resource_servers,create:resource_servers,update:resource_servers,read:roles,create:roles,update:roles,update:tenant_settings,read:connections,update:connections"
```

#### Enable Resource Parameter Compatibility Profile

In the [Auth0 Dashboard](https://manage.auth0.com/dashboard/):

1. Navigate to **Settings** → **Advanced**
2. Enable the **Resource Parameter Compatibility Profile** toggle

#### Promote connections to domain-level

So third-party clients (MCP Inspector, Claude, ChatGPT) can use them:

```bash
# List connections to find their IDs
auth0 api get connections

# Promote a connection (e.g. username-password database)
auth0 api patch connections/YOUR_CONNECTION_ID --data '{"is_domain_connection": true}'
```

### 2. Create an API to represent your MCP server

```bash
auth0 api post resource-servers --data '{
  "identifier": "http://localhost:3000/",
  "name": "MCP Tools API",
  "signing_alg": "RS256",
  "token_dialect": "rfc9068_profile_authz",
  "enforce_policies": true,
  "scopes": [
    {"value": "tool:whoami", "description": "Access user info and profile tools"},
    {"value": "tool:greet", "description": "Access the greeting tool"}
  ]
}'
```

> The `rfc9068_profile_authz` dialect includes the `permissions` claim in access tokens so you can do scope-based authorization in tools.

### 3. (Optional) Create roles and assign permissions

```bash
auth0 roles create --name "Tool Administrator" --description "Access to all MCP tools"
auth0 roles permissions add YOUR_ROLE_ID --api-id "http://localhost:3000/" --permissions "tool:whoami,tool:greet"
auth0 users roles assign USER_ID --roles ROLE_ID
```

### 4. Configure environment variables

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

Required:

```bash
MCP_USE_OAUTH_AUTH0_DOMAIN=your-tenant.us.auth0.com
MCP_USE_OAUTH_AUTH0_AUDIENCE=http://localhost:3000/
```

### 5. Install and run

```bash
pnpm install
pnpm dev
```

This starts:

- The MCP server on port **3000**
- The MCP Inspector at <http://localhost:3000/inspector>

## Try it out

1. Open <http://localhost:3000/inspector>
2. Connect to `http://localhost:3000/mcp`
3. Complete the Auth0 login flow
4. Call the available tools

## Available tools

| Tool                     | Required scope    | Description                                                |
| ------------------------ | ----------------- | ---------------------------------------------------------- |
| `get-user-info`          | `tool:whoami`     | Returns user details extracted from the JWT                |
| `get-auth0-user-profile` | `tool:whoami`     | Fetches the full user profile from Auth0's `/userinfo` endpoint |

## How the OAuth flow works

This template uses Auth0's DCR-direct flow — MCP clients communicate directly with Auth0 for all OAuth operations. Your server only publishes metadata and verifies tokens.

```
MCP Client ──(1) GET /.well-known/oauth-protected-resource ─▶ MCP Server
MCP Client ──(2) GET /.well-known/oauth-authorization-server (passthrough) ─▶ MCP Server ─▶ Auth0
MCP Client ──(3) Dynamic Client Registration ─▶ Auth0
MCP Client ──(4) PKCE authorization + token exchange ─▶ Auth0
MCP Client ──(5) MCP request + Bearer <token> ─▶ MCP Server (verifies JWT via JWKS)
```

## Deploy

```bash
npx mcp-use deploy
```

Or build a Docker image:

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build
CMD ["pnpm", "start"]
```

## Adding more scopes

1. Add the scope to your Auth0 API:

   ```bash
   auth0 api patch resource-servers/YOUR_API_ID --data '{
     "scopes": [{"value": "tool:custom", "description": "Your new tool"}]
   }'
   ```

2. Assign it to a role:

   ```bash
   auth0 roles permissions add ROLE_ID --api-id "http://localhost:3000/" --permissions "tool:custom"
   ```

3. Check the scope in your tool:

   ```typescript
   server.tool({ name: "my-tool", description: "..." }, async (_args, ctx) => {
     if (!ctx.auth.scopes?.includes("tool:custom")) {
       return error("Insufficient permissions: requires tool:custom");
     }
     // ...
   });
   ```

## Troubleshooting

- **"JWT verification failed"** — check `MCP_USE_OAUTH_AUTH0_AUDIENCE` matches your API identifier exactly (trailing slash matters), and `MCP_USE_OAUTH_AUTH0_DOMAIN` is the tenant domain (no `https://` prefix).
- **"Insufficient permissions"** — call `verify-token` to inspect the token's permissions; ensure the user's role includes the required scope; re-authenticate to pick up new permissions.
- **CORS errors in browser clients** — add the inspector's origin to your Auth0 Allowed Web Origins.

## Learn more

- [Auth0 MCP Authorization Guide](https://auth0.com/ai/docs/mcp/get-started/authorization-for-your-mcp-server)
- [mcp-use docs](https://mcp-use.com/docs)
- [MCP Authorization spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)
- [RFC 9068 — JWT Profile for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc9068)

## License

MIT
