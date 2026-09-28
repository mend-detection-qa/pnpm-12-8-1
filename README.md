# workspace-peer-dep-injection

Probe pattern: `workspace-peer-dep-injection`
pnpm version under test: **12.8.1**
Lockfile version: **9.0**
Generated: 2026-09-28

## Feature exercised

This probe exercises two interacting pnpm 12.8.1 behaviors that
affect Mend UA's `PnpmParserV9Impl`:

1. **Peer dependency snapshot suffixes in v9 lockfile.** When a
   package resolves under multiple peer-dep contexts, pnpm v9
   records each distinct peer combination as a snapshot key with a
   parenthesized suffix (e.g.
   `@testing-library/react@14.3.1(react@18.3.1)(react-dom@18.3.1(react@18.3.1))`).
   pnpm 12.8.1 refines this suffix for deeply nested peer chains
   where the peer itself is a peer-resolved package
   (`react-dom@18.3.1(react@18.3.1)` as a peer of
   `@testing-library/react`). Mend UA must strip the peer suffix
   from the snapshot key to identify the base package and version.

2. **Workspace package injection (`injected: true`).** The
   `packages/app` workspace declares `@probe/ui` with
   `dependenciesMeta: { "@probe/ui": { "injected": true } }`.
   This causes pnpm to hard-link (not symlink) the local package
   into `node_modules/`, which is needed when lifecycle scripts
   require a physically present node_modules layout. The lockfile
   records the workspace dep as `link:../ui` in `importers`,
   regardless of injection mode. Mend UA must detect this as
   `source: local`, not `source: registry`, even though the
   `node_modules/` layout looks like a registry install.

## Workspace layout

```
workspace-peer-dep-injection-20260928-000000/
├── package.json              root (private, no own deps)
├── pnpm-workspace.yaml       declares packages/*
├── pnpm-lock.yaml            v9 lockfile with peer-suffix snapshots
├── .whitesource              pins pnpm 12.8.1 + node 20.18.0
├── README.md
├── expected-tree.json
└── packages/
    ├── ui/
    │   └── package.json      @probe/ui — React component library
    │                         deps: clsx
    │                         peerDeps: react, react-dom
    │                         devDeps: @testing-library/react
    └── app/
        └── package.json      @probe/app — consumer app
                              deps: @probe/ui (injected), react,
                                    react-dom, hono
```

## Expected dependency tree

### Root importer (`.`)

No dependencies — the root is a private workspace coordinator.

### `packages/ui` importer (`@probe/ui@0.1.0`)

Direct dependencies:
- `clsx@2.1.1` — `source: registry`, `group: main`

Dev dependencies (direct):
- `@testing-library/react@14.3.1` — `source: registry`,
  `group: dev`.
  Mend must strip the peer suffix from the snapshot key
  `@testing-library/react@14.3.1(react@18.3.1)(react-dom@18.3.1(react@18.3.1))`
  and report version `14.3.1`, not the suffixed form.

Transitives of `@testing-library/react`:
- `@testing-library/dom@9.3.4` — `source: registry`
  - `@adobe/css-tools@4.4.1`
  - `aria-query@5.3.2`
  - `dom-accessibility-api@0.5.16`
  - `lz-string@1.5.0`
  - `pretty-format@27.5.1`
    - `react-is@17.0.2`

### `packages/app` importer (`@probe/app@0.1.0`)

Direct dependencies:
- `@probe/ui@0.1.0` — `source: local` (injected workspace pkg).
  Must NOT appear as `source: registry` even though injected into
  `node_modules/` via hard-link instead of symlink.
- `hono@4.6.3` — `source: registry`, `group: main`
- `react@18.3.1` — `source: registry`, `group: main`
  - Transitive: `loose-envify@1.4.0`
    - Transitive: `js-tokens@4.0.0`
- `react-dom@18.3.1` — `source: registry`, `group: main`
  Mend must strip peer suffix from snapshot key
  `react-dom@18.3.1(react@18.3.1)` and report version `18.3.1`.
  - Transitive: `scheduler@0.23.2`

## Mend failure modes targeted

1. Injected dep (`@probe/ui`) detected as `source: registry`
   instead of `source: local` (filesystem scan confused by
   hard-link in node_modules).
2. Peer suffix included in the reported version string (e.g.
   version reported as
   `14.3.1(react@18.3.1)(react-dom@18.3.1(react@18.3.1))` instead
   of `14.3.1`).
3. Snapshot key with nested peer suffix not matched; transitive
   deps of `@testing-library/react` silently dropped.
4. `react-dom` version reported as
   `18.3.1(react@18.3.1)` instead of `18.3.1`.

## Mend UA resolver notes

Mend's `PnpmLockCollector` with `PnpmParserV9Impl`:
- Reads `importers` section for workspace package lists.
- Reads `snapshots` section for resolved dependency graphs.
- Snapshot keys have format `<name>@<version>(<peer>@<pver>)...`
  — the parser must split at the first `(` to get name+version.
- For workspace cross-references, `link:` protocol in the
  `importers` section yields `source: local`.
- Injected packages still use `link:` in the lockfile importer
  section (the injection is a node_modules layout detail, not a
  lockfile format change).

Source: UA resolver section "6. pnpm Resolver"
Resolver SHA: 351915ae1d53c6db20db0a5d65b637fe7899d7ea
Fetched at: 2026-09-28T18:01:20+00:00

## Mend config

**Bucket A** — `js-pnpm` has no dynamic PM version detection from
the manifest. This probe ships `.whitesource` pinning:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "pnpm": "12.8.1",
      "node": "20.18.0"
    }
  }
}
```

No `whitesource.config` is present, so `configMode` is `"AUTO"`.
