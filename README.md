# web_mcp — SearXNG MCP server

Docker Compose deployment of [aicrafted/searxng-mcp](https://github.com/aicrafted/searxng-mcp)
backed by a private [SearXNG](https://github.com/searxng/searxng) instance, giving an MCP
client (e.g. Open WebUI) a `web_search` tool over streamable HTTP.

Deployed on `jf` (192.168.8.204) at `/srv/mcp/searxng-mcp`.

## Bring up

```
docker compose up -d
```

First run generates `searxng-config/settings.yml` from the image's template (not tracked in
this repo — see `.gitignore`). Two manual edits are required after the first run:

1. Enable JSON output (the MCP server reads SearXNG's JSON API, not HTML):

   ```yaml
   search:
     formats:
       - html
       - json
   ```

2. Disable the bot-detection limiter, since there's no reverse proxy in front of SearXNG to
   set `X-Forwarded-For`/`X-Real-IP` (without this, every search returns a 403):

   ```yaml
   server:
     limiter: false
   ```

Then `docker compose restart searxng`.

## Network exposure

- SearXNG itself is not published to the host — only reachable from the `searxng-mcp`
  container over the compose-internal network (`SEARXNG_URL=http://searxng:8080`).
- The MCP server is published on the host at `MCP_PORT` (8766), bound to `0.0.0.0`.
- This server has no bearer-token auth (unlike `ghidra-mcp` on this same host) — DNS-rebinding
  protection via `MCP_ALLOWED_HOSTS`/`MCP_ALLOWED_ORIGINS` is the only application-level check.
- Network access is restricted at the host firewall level to the Unraid box
  (192.168.8.152) only. **Important:** Docker-published ports bypass `iptables`'s `INPUT`
  chain (and therefore `ufw`, which only manages `INPUT`) — the restriction has to live in
  the `DOCKER-USER` chain instead:

  ```
  iptables -I DOCKER-USER -p tcp --dport 8766 ! -s 192.168.8.152 -j DROP
  ```

  This rule is host-level config, not part of this repo, and must be re-applied (or
  persisted via `netfilter-persistent`) if the host's firewall rules are ever rebuilt.

## MCP client config

Open WebUI → Admin Settings → External Tools → add a connection, type **MCP (Streamable
HTTP)**, URL `http://192.168.8.204:8766/mcp`, no auth.
