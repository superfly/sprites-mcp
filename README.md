# Sprites MCP Server

The official [Model Context Protocol](https://modelcontextprotocol.io) server for
[**Sprites**](https://sprites.dev) — sandboxed, checkpointable compute environments you can
create, run commands in, and manage services on, all from your AI agent.

This is a **remote MCP server**. There's nothing to install and run locally: your MCP client
connects directly over HTTPS to the hosted endpoint and authenticates with OAuth. This
repository is the public home for the server — its listing manifest, connection instructions,
and tool reference. The server itself is operated by [Fly.io](https://fly.io) as part of the
Sprites platform.

- **Endpoint:** `https://sprites.dev/mcp`
- **Transport:** Streamable HTTP (JSON-RPC 2.0)
- **Auth:** OAuth 2.1 (`sprites:read` / `sprites:write` scopes)
- **Registry:** [`dev.sprites/mcp`](https://registry.modelcontextprotocol.io/v0/servers?search=dev.sprites/mcp) in the official MCP registry
- **Docs:** https://docs.sprites.dev/integrations/remote-mcp/

[![Add sprites MCP server to Cursor](https://cursor.com/deeplink/mcp-install-dark.svg)](cursor://anysphere.cursor-deeplink/mcp/install?name=sprites&config=eyJ1cmwiOiJodHRwczovL3Nwcml0ZXMuZGV2L21jcCJ9)

## What is Sprites?

A Sprite is an isolated, persistent compute environment. You can create one, execute commands
inside it, run long-lived services, snapshot its full state as a checkpoint and restore later,
and control its outbound network policy — all programmatically. The MCP server exposes these
capabilities as tools so an AI agent can operate Sprites on your behalf.

Learn more at [sprites.dev](https://sprites.dev) and [docs.sprites.dev](https://docs.sprites.dev).

## Connecting

Because it's a remote server, you point your client at the URL and complete an OAuth login in
the browser the first time. No package install, no local process.

### Claude Code

```bash
claude mcp add --transport http sprites https://sprites.dev/mcp
```

Then run `/mcp` inside Claude Code to complete the OAuth login.

### Cursor

Click the **Add sprites MCP server to Cursor** button above, or add it manually to
`~/.cursor/mcp.json` (global) or `.cursor/mcp.json` (per project):

```json
{
  "mcpServers": {
    "sprites": {
      "url": "https://sprites.dev/mcp"
    }
  }
}
```

Cursor will prompt you to authorize via OAuth on first use.

### Claude Desktop / other HTTP-native clients

Add a remote server pointing at `https://sprites.dev/mcp`. In a generic MCP client config that
supports remote servers, that looks like:

```json
{
  "mcpServers": {
    "sprites": {
      "url": "https://sprites.dev/mcp"
    }
  }
}
```

You'll be prompted to authorize via OAuth on first use.

### Clients that only speak stdio

Some older MCP clients can only launch a local command over stdio and can't dial a URL directly.
For those, the generic [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) adapter bridges
stdio to the remote endpoint (it handles the OAuth flow for you):

```json
{
  "mcpServers": {
    "sprites": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://sprites.dev/mcp"]
    }
  }
}
```

`mcp-remote` is a third-party, off-the-shelf tool — it is **not** the Sprites server, just a
transport shim for clients that need one.

## Authentication

The server implements OAuth 2.1 with the metadata-discovery and dynamic-client-registration
endpoints MCP clients expect, so most clients configure themselves automatically once you
provide the URL:

| Endpoint | Purpose |
| --- | --- |
| `GET /.well-known/oauth-protected-resource` | Protected Resource Metadata (RFC 9728) |
| `GET /.well-known/oauth-authorization-server` | Authorization Server Metadata (RFC 8414) |
| `POST /oauth/register` | Dynamic Client Registration (RFC 7591) |
| `GET`/`POST /oauth/authorize` | Authorization endpoint |
| `POST /oauth/token` | Token endpoint |

**Scopes:** `sprites:read` (list/get operations) and `sprites:write` (create/update/destroy and
command execution).

**Restricted tokens.** During the consent flow you can restrict a token — limit it to a sprite
name prefix, cap the total number of sprites it may create, scope it to a label, or set an
expiry. Restrictions are enforced on every call.

## Tools

18 tools, grouped below. Access is **read** (safe, read-only) or **write** (creates, mutates, or
executes). Every sprite-scoped tool takes a required `sprite` parameter naming the target sprite.

### Sprite management

| Tool | Access | Description |
| --- | --- | --- |
| `list_sprites` | read | List sprites for the authenticated organization. |
| `create_sprite` | write | Create a new sprite. |
| `destroy_sprite` | write | Destroy a sprite. |

### Command execution

| Tool | Access | Description |
| --- | --- | --- |
| `exec` | write | Run a command inside a sprite (non-TTY HTTP exec). |
| `exec_list` | read | List active exec sessions. |
| `exec_kill` | write | Kill an exec session by ID. |

### Checkpoints

| Tool | Access | Description |
| --- | --- | --- |
| `checkpoint_create` | write | Snapshot the current sprite state. |
| `checkpoint_list` | read | List checkpoints. |
| `checkpoint_get` | read | Get details of a specific checkpoint. |
| `checkpoint_restore` | write | Restore the sprite to a checkpoint. |

### Services

| Tool | Access | Description |
| --- | --- | --- |
| `service_list` | read | List configured services and their state. |
| `service_get` | read | Get details of a specific service. |
| `service_create` | write | Create or update a service definition. |
| `service_start` | write | Start a service. |
| `service_stop` | write | Stop a running service. |
| `service_logs` | read | Stream logs for a service. |

### Network policy

| Tool | Access | Description |
| --- | --- | --- |
| `policy_network_get` | read | Get the current outbound network policy. |
| `policy_network_update` | write | Update the outbound network policy. |

### Parameters

Common: every sprite-scoped tool requires `sprite` (string) — the target sprite name.

- **`list_sprites`** — `prefix` (string, optional), `max_results` (integer, optional),
  `continuation_token` (string, optional).
- **`create_sprite`** — `name` (string, **required**), `wait_for_capacity` (boolean, optional).
- **`destroy_sprite`** — `name` (string, **required**).
- **`exec`** — `cmd` (string, **required**, repeatable for command + args), `path` (string,
  optional), `stdin` (boolean, optional), `env` (string `KEY=VALUE`, optional, repeatable),
  `dir` (string, optional).
- **`exec_kill`** — `session_id` (string, **required**), `signal` (string, optional, default
  `SIGTERM`), `timeout` (string, optional, default `10s`).
- **`checkpoint_create`** — `comment` (string, optional).
- **`checkpoint_get`** — `checkpoint_id` (string, **required**, e.g. `v7`).
- **`checkpoint_restore`** — `checkpoint_id` (string, **required**).
- **`policy_network_update`** — `rules` (array, **required**; each rule has `domain`, `action`,
  and optional `include`).
- **`service_get` / `service_start` / `service_stop` / `service_logs`** — `service_name`
  (string, **required**). `service_start`/`service_stop` take an optional `duration`/`timeout`;
  `service_logs` takes optional `lines` (integer) and `duration` (string).
- **`service_create`** — `service_name` (string, **required**), `cmd` (string, **required**),
  `args` (array, optional), `needs` (array, optional), `http_port` (integer, optional),
  `duration` (string, optional).

The authoritative, always-current tool schemas are served by the live server via the MCP
`tools/list` method.

## Registry & discovery

This server is published to the official MCP registry as `dev.sprites/mcp`. Directories that
ingest the registry (Smithery, Glama, the GitHub MCP registry, and others) pick it up
automatically. The manifest lives in [`server.json`](./server.json).

## About this repository

This repo contains public listing metadata and documentation only. The MCP server is
implemented as part of the Sprites platform and served remotely at `https://sprites.dev/mcp` —
there is no server code to run here.

## License

[MIT](./LICENSE) © Fly.io, Inc.
