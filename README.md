# Team-governed Fentaris proxy

A runnable TypeScript example of a shared MCP endpoint with API-key authentication,
group policies, remote tools, local tools, and structured JSON logs. Built with
[Fentaris](https://github.com/Fentaris/fentaris).

This project is maintained separately from the Fentaris SDK repository. It uses
published `@fentaris/core` and `@fentaris/cli` packages and has its own lockfile.

## What you will run

The proxy listens on `http://127.0.0.1:4100/mcp` and combines two namespaces:

- `specification`: a public remote MCP server at `https://mcp.specification.website/mcp`.
- `workspace`: local tools implemented in `src/index.ts` with `app.local(...)`.

| API-key user | Group | Visible and callable tools |
| --- | --- | --- |
| `reader` | `readers` | `specification__*` and `workspace__status` |
| `maintainer` | `maintainers` | Reader access plus `workspace__release_notes` |

Policy applies to both tool discovery and execution. Calling a hidden tool
directly is denied before its handler runs. Logs include subject and group tags.

## Prerequisites

- Node.js 24 or newer.
- pnpm 11.
- Network access for dependency installation and the optional public upstream.

The local `workspace` tools work even when the remote upstream is unavailable.

## Quick start

```bash
git clone https://github.com/Fentaris/team-governed-proxy.git
cd team-governed-proxy
pnpm install --frozen-lockfile
pnpm build
pnpm check
pnpm run doctor
pnpm exec fentaris auth api-key add reader --generate --non-interactive
pnpm exec fentaris auth api-key add maintainer --generate --non-interactive
pnpm dev
```

Save both generated client keys when printed. The following sections explain
credential storage and how to verify access as each user.

## Install and validate

```bash
pnpm install --frozen-lockfile
pnpm build
pnpm check
pnpm run doctor
```

All four commands should exit successfully before provisioning local identity.

## Provision API-key identities

Let the CLI generate a project-local encryption key in the ignored `.env` on
the first credential write:

```bash
pnpm exec fentaris auth api-key add reader --generate --non-interactive
pnpm exec fentaris auth api-key add maintainer --generate --non-interactive
```

Save each generated API key when it is printed; Fentaris stores only its hash.
For the smoke tests below, set `READER_API_KEY` or `MAINTAINER_API_KEY` in the
client shell without committing either value.

The encrypted `.fentaris/credentials.enc.json` file and encryption key are
local state and must not be committed. The committed
`.fentaris/secrets.manifest.json` contains schema only.

## Start the proxy

```bash
pnpm dev
```

Expected startup output includes:

```txt
Proxy ready
Listening on: http://127.0.0.1:4100/mcp
```

In a second shell, export the same client API key used for the curl tests so
doctor can authenticate. The project script loads `FENTARIS_AUTH_KEY` from
`.env` automatically:

```bash
export FENTARIS_API_KEY="$READER_API_KEY"
pnpm exec fentaris doctor --runtime --non-interactive
```

`FENTARIS_AUTH_KEY` unlocks the local encrypted store; `FENTARIS_API_KEY` is
the raw client key sent as `x-fentaris-api-key`. Without the latter, runtime
probing returns HTTP 401 on this example.

Expected result: the MCP initialize check passes for
`http://127.0.0.1:4100/mcp`.

## Test an authenticated MCP session

Initialize as the reader and save the returned session header:

```bash
curl -sS -D reader-headers.txt -o reader-initialize.json \
  -X POST http://127.0.0.1:4100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "x-fentaris-api-key: $READER_API_KEY" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"curl","version":"1"}}}'

READER_SESSION="$(grep -i '^mcp-session-id:' reader-headers.txt | tr -d '\r' | cut -d' ' -f2)"
```

Send the initialized notification and list visible tools:

```bash
curl -sS -X POST http://127.0.0.1:4100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "x-fentaris-api-key: $READER_API_KEY" \
  -H "mcp-session-id: $READER_SESSION" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -sS -X POST http://127.0.0.1:4100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "x-fentaris-api-key: $READER_API_KEY" \
  -H "mcp-session-id: $READER_SESSION" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

The reader result includes `workspace__status` and hides
`workspace__release_notes`. Remote `specification__*` tools appear when the
public upstream is reachable.

Calling the hidden maintainer tool directly as the reader is denied before its
local handler runs:

```bash
curl -sS -X POST http://127.0.0.1:4100/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "x-fentaris-api-key: $READER_API_KEY" \
  -H "mcp-session-id: $READER_SESSION" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"workspace__release_notes","arguments":{}}}'
```

Expected result: the MCP tool result has `isError: true`, with Fentaris error
code `-32030` and denial reason `not-permitted` in `_meta.error`. Repeat the
session with `MAINTAINER_API_KEY`; the maintainer tool is listed and returns:

```txt
Release notes are visible to maintainers only.
```

You can also use MCP Inspector:

```bash
npx @modelcontextprotocol/inspector
```

Point it at `http://127.0.0.1:4100/mcp` and set the
`x-fentaris-api-key` request header.

## Project layout

| File | Purpose |
| --- | --- |
| `src/index.ts` | Users, groups, policies, upstream, local tools, and log tags |
| `fentaris.json` | Endpoint, entrypoint, and local auth directory |
| `.fentaris/secrets.manifest.json` | Committed upstream-secret schema (empty for this example) |
| `pnpm-lock.yaml` | Locked dependencies for reproducible installs |
| `tsconfig.json` | Strict TypeScript configuration |

`pnpm dev` runs the TypeScript entrypoint; `pnpm build` compiles it and `pnpm start`
runs the compiled application. Both start commands load the local `.env` file.

## Troubleshooting

- **HTTP 401:** send the raw client API key in `x-fentaris-api-key`.
  `FENTARIS_AUTH_KEY` unlocks the credential store; it is not a client API key.
- **Credentials cannot be decrypted:** use the encryption key that created the
  local store. Check `.env` or an explicitly exported `FENTARIS_AUTH_KEY`.
- **Remote tools are missing:** check connectivity to the public upstream.
  Verify `workspace__status` first to test the local proxy independently.
- **Port 4100 is occupied:** stop the other process or change `port` in `fentaris.json`.
- **Doctor warns about API-key hashes not listed in the manifest:** the manifest
  describes upstream secrets; the API-key hashes live separately in the encrypted
  auth store. This example has no upstream credentials.

## Scope and extensions

This example demonstrates two groups, one remote upstream, local tools, API-key
identity, policy filtering, and JSON logging. The curl checks above are manual.
Rate limits, Telegram approvals, three remote upstreams, and automated smoke
checks are future extensions. Configure network controls before making the
localhost endpoint accessible to other machines.

## Documentation and license

- [Fentaris documentation](https://fentaris.mintlify.app)
- [Team example walkthrough](https://fentaris.mintlify.app/examples/team-governed-proxy)
- [Fentaris SDK source](https://github.com/Fentaris/fentaris)

MIT licensed; see [LICENSE.txt](LICENSE.txt).
