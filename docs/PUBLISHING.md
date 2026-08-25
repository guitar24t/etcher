Publishing Etcher
=================

This fork builds and publishes its own releases from GitHub Actions
(`.github/workflows/build.yml`). Nothing is published to balena's channels.

Cutting a release
-----------------

1. Bump `version` in `package.json` and the two matching `version` fields at
   the top of `npm-shrinkwrap.json`.
2. Add a section to `CHANGELOG.md`.
3. Commit, then tag and push:

   ```sh
   git tag -a v2.1.7 -m v2.1.7
   git push origin v2.1.7
   ```

Pushing a `v*` tag builds all four targets and publishes them to a GitHub
release along with `SHA256SUMS.txt`. The release job only runs if every
platform built, so a release is never partially populated.

| Target       | Runner            | Artifacts                |
| ------------ | ----------------- | ------------------------ |
| Linux x64    | `ubuntu-22.04`    | `.deb`, `.rpm`, `.zip`   |
| Windows x64  | `windows-2022`    | `Setup.exe`, `.zip`      |
| macOS arm64  | `macos-15`        | `.dmg`, `.zip`           |
| macOS x64    | `macos-15-intel`  | `.dmg`, `.zip`           |

Building locally
----------------

```sh
npm ci
npm run make
```

Artifacts land in `out/make`.

**Node 22 is required — not merely recommended.** `forge.sidecar.ts` packages
the `etcher-util` sidecar with `pkg --target node22.22.2`, so the sidecar
embeds a Node 22 runtime. `mountutils` is not an N-API addon and is rebuilt
against whichever Node runs the build, so building on Node 20 (ABI 115) or
Node 24 (ABI 137) produces an addon the Node 22 sidecar (ABI 127) cannot load.
The failure is silent at build time and only shows up as a sidecar that dies
the moment drive scanning starts. `engines` pins this.

Per-platform prerequisites:

- **Windows** — Visual Studio with the C++ workload. `winusb-driver-generator`
  compiles libwdi from source, and its `deps/embed.bat` invokes `embedder.exe`
  from the working directory, so the build fails with
  `'embedder.exe' is not recognized` if `NoDefaultCurrentDirectoryInExePath=1`
  is set in the environment. Unset it before building.
- **Linux** — `fakeroot`, `dpkg` and `rpm` for the packaging targets. Note that
  `rpmbuild` strips binaries by default, which corrupts the pkg-built sidecar;
  CI works around this with a `%__strip /usr/bin/true` macro in `~/.rpmmacros`.
- **macOS** — Xcode command line tools.

Signing
-------

Released builds are **unsigned**. `forge.config.ts` switches on
`NODE_ENV=production`:

- Unset (the default, and what CI uses) — macOS gets an ad-hoc signature, which
  is the minimum needed for a repackaged arm64 bundle to launch at all. Windows
  is not signed.
- `production` — signs with a Developer ID and notarizes via `notarytool`, and
  signs Windows with `signtool`. This requires `XCODE_APP_LOADER_EMAIL`,
  `XCODE_APP_LOADER_PASSWORD`, `XCODE_APP_LOADER_TEAM_ID`,
  `SM_CODE_SIGNING_CERT_SHA1_HASH` and `TIMESTAMP_SERVER`, plus the
  certificates themselves installed on the runner.

Because the published builds are unsigned, SmartScreen warns on Windows, and
macOS reports the app as damaged until the quarantine flag is cleared:

```sh
xattr -dr com.apple.quarantine /Applications/balenaEtcher.app
```

The release notes repeat this for anyone downloading a build.

Auto-updates
------------

`packageType` in `package.json` is `local`, which leaves `packageUpdatable`
false in `lib/gui/etcher.ts`. These builds therefore never check for updates,
and will not try to replace themselves with an upstream balena release. Keep it
that way unless you stand up an update feed of your own.
