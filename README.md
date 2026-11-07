# WebRedstone Minefort Edition 2.1 - Paper 26.2

A Minecraft Paper 26.2 plugin that controls a redstone block from a separately hosted HTML website using an outbound-only HTTPS Cloudflare Worker relay. No extra Minefort port, no RCON, and no Minecraft client mods.

## Changes in 2.1

- **Paper 26.2 target**, pinned to official `io.papermc.paper:paper-api:26.2.build.135-stable`.
- **Java 25** required to compile and run (Paper 26.2 also requires Java 25).
- `plugin.yml` declares `api-version: '26.2'` and plugin version `2.1.0`.
- Added `.github/workflows/build-paper-26-2.yml` to automatically build the *real* Paper-API JAR on GitHub Actions. No local Java installation required.
- Website design and endpoints remain the same; Cloudflare relay remains compatible. Website's version label changes to v2.1.

## IMPORTANT: Build status

This package contains **source, not a verified 26.2 JAR**. This environment has Java 21 and cannot download Paper's dependency, so the official Paper 26.2 compile and live-server checks could not be completed here. **Build the JAR with the provided GitHub Actions workflow (Java 25 + official Paper API) before installing.** If the workflow fails, do not deploy; inspect and address the build error. Cloudflare Worker unit tests can run without Paper and are described below.

Minefort support for outbound HTTPS plugin requests has not been demonstrated on a live server, and the hosting provider may not expose Paper 26.2/Java 25 as an available choice on every server plan. Confirm both before installing.

## Files

| File | Purpose |
|---|---|
| `website/index.html` | Complete website with CSS/JS, safe for GitHub Pages once the relay URL is configured |
| `relay/worker.js` | Cloudflare Worker API for authenticated ON/OFF actions and device polling |
| `src/main/java/com/craftyweb/minefort/WebRedstoneMinefortPlugin.java` | Paper plugin source |
| `src/main/resources/config.yml` | Default plugin settings |
| `src/main/resources/plugin.yml` | Plugin metadata and permissions |
| `pom.xml` | Paper 26.2 dependency and Java 25 Maven build |
| `.github/workflows/build-paper-26-2.yml` | One-click GitHub Actions JAR build |
| `tests/relay.test.mjs` | Unit tests for the relay |

## How to get the actual JAR (no Java install on your PC)

1. On GitHub create a **private** repository with a README. Unzip this project on your PC.
2. Upload **the contents** of this folder to the repository root, including `.github/workflows/build-paper-26-2.yml`. Keep the `src/`, `relay/`, `website/`, `tests/` folders and `pom.xml` directly under the root. Uploading only the ZIP itself will **not** run the workflow.
3. Go to the repository's **Actions** tab. If GitHub asks, enable workflows for this repository.
4. Select **Build WebRedstone for Paper 26.2** â†’ **Run workflow** â†’ choose `main` â†’ **Run workflow**. It will also run after pushes to `main`.
5. Open the completed run. Verify the Java build shows success and the **Test Cloudflare relay** step passes.
6. In the run's **Artifacts** section, download `WebRedstone-Minefort-Paper26.2` and extract it. It contains `WebRedstone-Minefort-2.1.0-Paper26.2.jar`.
7. If any step fails, the ZIP is not ready for installation. Read the action log; do not substitute the older experimental 2.0 JAR.

Alternative: In Codespaces with a **Java 25 JDK** and Maven installed, run `mvn -U clean package` in the directory containing `pom.xml`; then download `target/WebRedstone-Minefort-2.1.0-Paper26.2.jar`. Java 21 cannot build this release.

## Cloudflare setup (same as 2.0)

1. Generate **two separate random secret values** of 32 bytes or more; label them `ADMIN_KEY` and `DEVICE_KEY`. In Codespaces you can run twice: `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"`.
2. Deploy the whole `relay/worker.js` as a Cloudflare Worker. Create a Workers KV namespace (any display name) and bind it to the Worker under the exact binding name **`STATE`**.
3. Set Worker secrets `ADMIN_KEY` and `DEVICE_KEY` to the two different generated values. Set text variable `ALLOWED_ORIGIN` to the **origin** of your website, e.g. `https://YOUR-NAME.github.io`, not the URL path.
4. Deploy settings. Confirm `https://YOUR-WORKER.workers.dev/health` responds with `"ok": true`.
5. Edit only the `RELAY_URL` constant in `website/index.html` and replace `https://REPLACE-ME.workers.dev` with your exact deployed Worker URL. Never put either secret into HTML/JavaScript.
6. Host `index.html` on an HTTPS site (e.g. GitHub Pages). For GitHub Pages, create a public repo with only `index.html` at its root; enable Settings â†’ Pages â†’ Deploy from branch â†’ main/root. Keep the plugin repository private.

**Existing installation of 2.0?** Keep your already deployed Worker, KV namespace, two keys and published website. They still work with 2.1. You can optionally publish the updated v2.1 website label; no endpoint changes are required.

## Install/upgrade on Minefort

1. Back up your world. Paper warns that upgrading a world to 26.2 cannot be reversed by simply downgrading the server.
2. Confirm the server software is **Paper 26.2** and server Java runtime is **Java 25**. A client connecting with 26.2 does **not** mean the server software is 26.2.
3. Stop the Minefort server; remove the old `WebRedstone-Minefort` (v2.0) JAR from the plugins directory; keep existing `plugins/WebRedstone/config.yml` (don't delete its real keys/binding). Do not run both JARs together.
4. Upload the newly compiled `WebRedstone-Minefort-2.1.0-Paper26.2.jar` into the `plugins` folder. Start/restart the server.
5. On first installation, in `plugins/WebRedstone/config.yml` set:

```yaml
relay:
  url: 'https://YOUR-WORKER.workers.dev'
  device-key: 'YOUR_PRIVATE_DEVICE_KEY'
  poll-seconds: 5
  heartbeat-seconds: 300
```

6. Restart. In the console look for `WebRedstone v2.1.0` and `Outbound-only WebRedstone relay is enabled` (assuming configured). Check for errors about API versions, Java, HTTPS, or HTTP 401.
7. Join as OP. Look at a solid block with an empty space above it. `/wr bind` reserves the space above; `/wr bind <x> <y> <z>` reserves an empty space at specified coordinates in your world. Place a redstone lamp adjacent to the reserved space.
8. Run `/wr on`, `/wr off`, `/wr status`. Then log in on your public HTTPS website using `ADMIN_KEY` and toggle power from there.
9. Remain near the bound block while testing: the plugin **doesn't force-load chunks**, and a sleeping Minefort server won't execute requests until it is running.

## Security and limits

The public page cannot receive the private keys hardcoded in the file. The Worker enforces the ADMIN_KEY, separates it from DEVICE_KEY, requires the configured browser Origin for control requests, and accepts only ON/OFFâ€”never arbitrary console commands. Protect both keys; rotate them if exposed. The browser holds ADMIN_KEY in memory only while the tab remains active.

Cloudflare KV is eventually consistent, so updates may be delayed beyond the five-second poll interval. Minefort's outbound HTTPS policy must be tested with your actual server. Use a backup and a disposable redstone circuit for first-time testing.

## Tests

Relay: `node --test tests/relay.test.mjs` (Node 22+). GitHub Actions also runs this test before building the JAR. **A successful compilation is not proof of Minefort runtime compatibility**; test in a real Paper 26.2 world. See `TESTING.md`.
