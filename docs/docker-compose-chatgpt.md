# Docker Compose + ChatGPT Secure MCP Tunnel

This deployment runs Zotero MCP locally and connects it to ChatGPT through
OpenAI's outbound-only Secure MCP Tunnel. It deliberately uses the core image:
keyword search, metadata, collections, PDF/full-text reading, and Zotero
write tools are available. The upstream ChatGPT-compatible `search` / `fetch`
tools are enabled; `search` falls back to Zotero keyword matching because this
core image has no semantic dependencies or index.

## 1. Prepare the local tunnel settings

Create the ignored settings file and fill it with the Platform runtime key and
tunnel ID:

```bash
cp .env.tunnel.example .env.tunnel
$EDITOR .env.tunnel
```

The runtime key is used only by `tunnel-client`. It is not a Zotero key and is
not stored in the repository.

## 2. Build and start the MCP service

Keep Zotero Desktop running with its local API enabled. The Compose service
uses host networking and binds only to loopback port `18473`; the Zotero data
directory is mounted read-only at `/zotero` so the server can read
`zotero.sqlite` and `storage/` attachments.

```bash
docker compose build zotero-mcp
docker compose up -d zotero-mcp
```

Check the local library through the CLI image:

```bash
docker compose run --rm --no-deps -e ZOTERO_APP=cli zotero-mcp get collections
docker compose run --rm --no-deps -e ZOTERO_APP=cli zotero-mcp search PHM --json
```

## 3. Grant local write access

Zotero 10 supports local writes. Run this once; Zotero will display an
authorization dialog. Choose **Always Allow** so the key is stored in the
Compose volume and reused after restarts:

```bash
docker compose run --rm --no-deps zotero-mcp authorize-local
```

The container still sends writes to Zotero's local API at
`http://127.0.0.1:23119`; it does not write the read-only `/zotero` mount.

## 4. Start and inspect the tunnel

After creating and associating a tunnel in OpenAI Platform, start the official
tunnel client:

```bash
docker compose --env-file .env.tunnel up -d tunnel-client
docker compose --env-file .env.tunnel logs -f tunnel-client
docker compose --env-file .env.tunnel run --rm tunnel-client doctor --explain
```

The tunnel client needs outbound HTTPS to `api.openai.com:443`; it needs no
inbound port. Its local health/admin endpoint is loopback-only at
`http://127.0.0.1:18474/ui`.

## 5. Add the connection in ChatGPT

Open ChatGPT **Plugins**, choose **Create app**, select **Tunnel**, and select
the associated tunnel ID. Review the discovered tools before creating the app.
The local MCP URL is never entered into ChatGPT; it remains
`http://127.0.0.1:18473/mcp` inside the local trust boundary.

Test in this order:

1. List the PHM collection.
2. Search by a short title or author substring.
3. Read one known PDF/full-text item.
4. Create a clearly named temporary collection and verify it in Zotero.

Delete the temporary collection only after checking it is the test object.
If ChatGPT does not expose or execute write tools, keep the tunnel private and
record that product limitation instead of switching to a public proxy.
