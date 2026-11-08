# WebRedstone Minefort Edition 2.2 FAST - Paper 26.2 / Java 25

A faster and much clearer version of WebRedstone. **This is a source project, not an already built official Paper JAR.**

## Why the old version sometimes needed 30 seconds

WebRedstone 2.1 stored `desired` and `ack` in Cloudflare Workers KV. KV is eventually consistent across different Cloudflare regions; changes can take 60 seconds or more to propagate to places which recently read an old value. Minefort's plugin and the browser often connect from different regions. Polling KV every second would *not* fix the fundamental stale-read problem.

**2.2 FAST** uses one **SQLite-backed Cloudflare Durable Object**. The website and Minecraft both talk to the same strongly consistent state. The plugin polls every **2 seconds**, and the website refreshes faster while a command is pending. We aim for roughly 2â€“5 seconds in normal conditions (not a guaranteed deadline).

Cloudflare Durable Objects are supported on Cloudflare Workers Free plans with usage limits. At the default 2-second poll, one continuously running Minecraft server produces approximately 43,200 poll requests/day; 7.5-second idle website refreshes add approximately 11,520 requests/day per continuously open tab. Multiple tabs/servers can exceed the free tier. Check limits in your Cloudflare dashboard.

## What is in the ZIP

- `website/index.html` â€” Minecraft-styled website with clear ON, OFF, pending and confirmed states.
- `relay-fast/worker.js`, `router.js`, `room.js`, `state-core.js` â€” NEW secure fast relay.
- `relay-fast/wrangler.jsonc` â€” sets Worker name `webredstone-fast`, allowed website origin and SQLite Durable Object `ROOM`. This automatically provisions the binding through a Wrangler deployment.
- `relay-fast/package.json` â€” Wrangler tool for deployment.
- `src/main/java/...` â€” updated Paper plugin, keeps `/wr bind`, `/wr status`, `/wr on`, `/wr off`.
- `.github/workflows/build-paper-26-2.yml` â€” official Paper 26.2 / Java 25 build workflow.
- `tests/fast-relay.test.mjs` â€” new relay tests, alongside previous KV relay tests.
- `UPGRADE-STEPS.md` â€” complete step-by-step upgrade and rollback.

## VERY IMPORTANT: preserve your working 2.1 while upgrading

Do **not** edit or delete your current Cloudflare Worker `webredstone` or its KV namespace. The new relay deploys as a **second Worker** called `webredstone-fast`, so you can fall back if anything goes wrong. Its public URL should be:

`https://webredstone-fast.creatorwes.workers.dev`

You can use the **same** existing ADMIN_KEY and DEVICE_KEY as the old Worker, but they need to be entered again as secrets on the **new** Worker. Never place either key in GitHub files or in `website/index.html`.

See `UPGRADE-STEPS.md` for the steps.

## Configuration after upgrading

In Minefort, edit `plugins/WebRedstone/config.yml` (preserve your existing `block:` section):

```yaml
relay:
  url: 'https://webredstone-fast.creatorwes.workers.dev'
  device-key: 'YOUR_EXISTING_PRIVATE_DEVICE_KEY'
  poll-seconds: 2
  heartbeat-seconds: 120
```

Your GitHub Pages `index.html` needs the same `RELAY_URL` value; the website does NOT contain the DEVICE_KEY. The website asks for ADMIN_KEY when you sign in.

Use the new `/health` endpoint to confirm the Worker:
`https://webredstone-fast.creatorwes.workers.dev/health`
Expected: JSON with `"ok": true`, `"storage": "durable-object"`.

## Tests performed locally

- Node.js 22: **35/35** combined old/new relay tests passed.
- New website JavaScript and Worker source passed JavaScript syntax checks.
- Not tested live on your Cloudflare account, GitHub Actions runner or Minefort server.
- The installable Paper JAR must be compiled via GitHub Actions before you upload it.

## Safety design

Website ADMIN_KEY authenticates ON/OFF changes; device DEVICE_KEY authenticates polling and acknowledgments. Auth secrets are compared using hashed constant-length digests. Only the configured browser `ALLOWED_ORIGIN` can post commands. Server plugin never runs arbitrary website commands; it only places/removes its own reserved redstone block in an already-loaded chunk on the main server thread. No plugin HTTP port is opened.
