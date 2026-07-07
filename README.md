# telosmud-content

The **external, versioned content store** for [TelosMUD](https://github.com/double-nibble/telosMUD) (engine issue #212). Content packs live here as the authoring source of truth; a git tag is a **published version** that a running deployment pulls with `telos-pull` into Postgres, from where the world serves it.

The engine's embedded `demo` pack is a *test fixture* and stays in the engine binary — it is **not** here. This repo holds real, deployable content.

## Layout

```
packs/<name>/…       one directory tree per content pack (pack.yaml + zones/, abilities.yaml, …)
manifest.yaml        the version descriptor: version, content_hash, packs[]  (generated — see below)
.gitattributes       * text=auto eol=lf  — LF everywhere so the content_hash is reproducible
```

## How a version is published

1. Edit content under `packs/`.
2. Bump `version:` and regenerate `manifest.yaml`:

   ```sh
   # build telos-pull from an engine checkout (the engine module has a go.mod replace, so
   # `go install` won't work — build it from a clone):
   git clone --depth 1 https://github.com/double-nibble/telosMUD.git /tmp/engine
   (cd /tmp/engine && go build -o /tmp/telos-pull ./cmd/telos-pull)

   /tmp/telos-pull --emit-manifest --dir . --manifest-version v0.2.0
   ```

3. Open a PR. **`validate`** CI runs `telos-pull --check` — it verifies the manifest's `content_hash` matches the tree, that `packs[]` matches the directories, that every pack parses, and that no pack ships a reserved `core:` ref. A stale manifest (edited packs without regenerating) fails the check.
4. Merge to `main`. **`publish`** CI re-validates and tags the commit with `manifest.version`. That tag is the published version.

## How a deployment pulls a version

`telos-pull` (from the engine) imports a published version into Postgres atomically (pruning packs a prior version dropped, stamping a monotonic content version) and broadcasts a hot-reload so running shards pick it up:

```sh
TELOS_CONTENT_URL=https://github.com/double-nibble/telosmud-content.git \
TELOS_CONTENT_VERSION=v0.2.0 \
TELOS_POSTGRES_DSN=postgres://… \
TELOS_NATS_URL=nats://… \
telos-pull
```

The world then serves exactly this version's packs (manifest-driven). A private content repo passes a PAT via `TELOS_CONTENT_TOKEN` (never embed credentials in the URL).

## Constraints

- **Pack refs must be globally disjoint.** Most engine tables key rows by `ref` alone (globally unique), so two packs must not share a zone/room/item/mob ref — the atomic import fails (fail-safe) on a collision. Namespace refs per pack.
- **Cross-pack exits are not supported** — a room's exit must target a room in the same pack.
- **ASCII pack paths.** Non-ASCII filenames can normalize differently across hosts (macOS NFD vs Linux NFC) and diverge the content hash. Keep `packs/` paths ASCII.
