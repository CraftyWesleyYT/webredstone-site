# Validation record — WebRedstone Minefort Edition 2.1 (Paper 26.2)

## Automated checks

- Cloudflare Worker security/command tests: `node --test tests/relay.test.mjs`.
- Website JavaScript syntax: extract the inline script and test with `node --check`.
- Maven build configuration: version 2.1.0, `maven.compiler.release=25`, official pinned Paper dependency `26.2.build.135-stable`, and `api-version: 26.2`.
- GitHub Actions workflow provisions Temurin Java 25, runs relay tests, builds with official Paper API, and attaches the JAR for download.

## Not tested in this workspace

- Official Java 25 Maven compilation (only Java 21 and no Maven/remote repository access were available).
- Running the resulting JAR on Paper 26.2 or Minefort.
- Minefort's outgoing HTTPS permissions, plugin upload/build restrictions.
- Cloudflare production KV behavior and remote website deployed with real keys.

**Do not treat the package as a verified working JAR.** Obtain a JAR from a successful GitHub Actions build and do a live test on a backed-up Paper 26.2 server.
