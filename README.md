# torTV plugin registry (official)

The **official, curated source** for torTV plugins. This repo is intentionally
**public** so torTV boxes can fetch it over **anonymous HTTPS raw** — no
credentials, ever. Trust comes from the **signature**, not from the host: the box
verifies `registry.toml.sig` against the Ed25519 public key baked into the binary
(`PLUGIN_PUBKEY`), then verifies each plugin's `sha256` before installing it into
the WASM sandbox under user-approved capability grants.

> This repo must stay **public**. Private-repo assets are rejected by the box by
> design (they would require an embedded credential).

## Layout

```
registry.toml                          # the index (signed)
registry.toml.sig                      # detached Ed25519 over registry.toml (verifies against PLUGIN_PUBKEY)
plugins/<id>/<version>/plugin.wasm     # the content-addressed plugin component (wasm32-wasip2)
```

## Index format (`registry.toml`)

One `[[plugin]]` table per published plugin + version:

```toml
[[plugin]]
id = "scraper-tmdb"
name = "TMDB"
kind = "MetadataScraper"        # Indexer | Resolver | MetadataScraper | SubtitleProvider | LibrarySource | ArtworkSource
version = "0.2.0"               # semver
wasm = "plugins/scraper-tmdb/0.2.0/plugin.wasm"
sha256 = "…"                    # integrity; the signed index vouches for these bytes
capabilities = ["http-fetch", "storage"]   # declared (a hint shown to the user at install)
author = "torTV"
description = "…"
homepage = "…"
```

## Trust chain (official)

1. Box GETs `registry.toml` + `registry.toml.sig` over HTTPS.
2. Verify `registry.toml.sig` against the baked `PLUGIN_PUBKEY` → trust the index.
3. Download the chosen `plugin.wasm` → assert its `sha256` matches the index.
4. Install into the sandbox; the user approves a capability **grant set**; the host
   enforces it at runtime (`http_fetch` traps if not granted).

## Publishing a plugin (project key holder)

1. Build the component: `cargo build -p <plugin> --target wasm32-wasip2 --release`.
2. Place it at `plugins/<id>/<version>/plugin.wasm` and compute its `sha256`.
3. Add/update the `[[plugin]]` entry in `registry.toml`.
4. **Re-sign the index** with the project signing key (detached Ed25519 over
   `registry.toml` → `registry.toml.sig`), using the `sign_plugin` helper from the
   torTV repo. The private key is held off-repo and never committed.
5. Commit + push. Boxes pick up the new/updated plugin on their next registry refresh.

## Third-party sources

Any third-party plugin source is a git repo with this **same layout**, just
**without** `registry.toml.sig`. Users add such sources in Settings → "Allow
third-party plugin sources" (off by default). Third-party plugins install
**unsigned**, guarded by the WASM sandbox + the user-approved capability grants.
