# AGENTS.md

This repo is the public listing for the hosted Sprites MCP server at `https://sprites.dev/mcp`.
It holds no server code, build, tests or CI: the server runs as part of the Sprites platform.

- `server.json`: the MCP Registry manifest for `dev.sprites/mcp`
- `.mcp.json`: a client config pointing at the hosted server
- `README.md`: connection instructions and the tool reference

## Editing `server.json`

- `title` and `description` are capped at 100 characters by the `$schema`.
- Bump `version` with every change. The registry rejects a version it already has.
- Validate before committing:

  ```sh
  uvx check-jsonschema --schemafile https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json server.json
  ```

## Publishing

Merging doesn't update the registry. Someone with the Ed25519 private key behind
`https://sprites.dev/.well-known/mcp-registry-auth` publishes by hand from this repo:

```sh
mcp-publisher login http --domain=sprites.dev --private-key-file=<key>
mcp-publisher publish
```

Check the result at `https://registry.modelcontextprotocol.io/v0/servers?search=dev.sprites/mcp`.

## Tool reference

The README's tool tables document the live server. Get the current tools from the server's MCP
`tools/list`, not from memory, when updating them.
