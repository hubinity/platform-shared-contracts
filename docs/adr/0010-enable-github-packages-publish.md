# ADR 0010 — Enable GitHub Packages publish (supersedes ADR 0006)

- **Status**: Accepted
- **Date**: 2026-08-11
- **Deciders**: Hubinity Platform team

## Context and Problem Statement

ADR 0006 deliberately deferred publishing: build-only CI, `<distributionManagement>`
commented out, no registry secrets. Consumers had to `mvn install` locally.
That repo is no longer greenfield-only — other services (and CI pipelines in
sibling repos) now need to resolve `contracts-*` and `@hubinity/tailwind-preset`
from GitHub Packages instead of a hand-built local `.m2`/`node_modules`. ADR
0006 already named this the "likely eventual target" and scoped it as a
one-PR change; this ADR is that follow-up.

## Decision Drivers

- Sibling repos' CI (11 other Hubinity repositories) cannot depend on a
  contributor's local `mvn install` — they need artifacts resolvable from a
  registry in a clean CI runner.
- Maven contracts (single reactor version, ADR 0005) and `tailwind-preset`
  (independent npm version, ADR 0008) release on different cadences and must
  not share a trigger.
- Publishing must stay opt-in per release, not on every push to `main` —
  ADR 0006's "avoid leaking half-baked versions" concern still holds.
- Credential handling should not require every consuming repo to mint its
  own PAT; a single org-level secret is simpler to rotate and audit.

## Considered Options

- **Publish on every push to `main`** — pros: always current; cons: floods
  the registry with unreviewed versions, reintroduces exactly what ADR 0006
  avoided. Rejected.
- **Manual `mvn deploy` / `npm publish` from a maintainer's machine** — pros:
  zero CI work; cons: relies on a human having the right local credentials
  and remembering to do it; not reproducible. Rejected.
- **Tag-triggered publish, one workflow per ecosystem** — pros: explicit,
  reviewable release step; reactor (Maven) and `tailwind-preset` (npm) tag
  independently, matching their independent versioning; the CI job double
  checks the tag against the manifest version before deploying, so a stale
  tag can't accidentally publish the wrong artifact. Accepted.

## Decision Outcome

**Chosen**: two tag-triggered workflows, both gated on the org-level
`GH_PACKAGES_TOKEN` secret (classic PAT, `write:packages` + `read:packages`
+ `delete:packages`, provisioned by an org owner — see `README.md` and
`tailwind-preset/README.md`):

- `.github/workflows/publish-maven.yml` — triggers on tags matching
  `v<version>` (e.g. `v0.1.0-SNAPSHOT`). Fails fast if the tag doesn't match
  `pom.xml`'s `<version>`, runs `mvn verify`, then `mvn deploy` against the
  now-uncommented `<distributionManagement>` block (`pom.xml`).
- `.github/workflows/publish-tailwind-preset.yml` — triggers on tags
  matching `tailwind-preset-v<version>` (e.g. `tailwind-preset-v0.1.0`).
  Fails fast if the tag doesn't match `tailwind-preset/package.json`'s
  `"version"`, then builds and `npm publish`es (registry already set via
  `publishConfig` in `package.json`).

`ci.yml` (build/verify on every push and PR) is unchanged — it still never
publishes.

## Consequences

- ✅ Sibling repos can `mvn install`/`npm install` from GitHub Packages in
  clean CI, no local `.m2`/`node_modules` bootstrap required.
- ✅ Publishing stays a deliberate, tagged action — a stray push to `main`
  can't leak an unstable version.
- ✅ One shared secret (`GH_PACKAGES_TOKEN`) instead of one per consumer.
- ⚠️ Tagging is now a two-step manual process (bump manifest version, then
  tag) — the version-match check exists specifically to catch the case where
  someone forgets step one.
- ⚠️ `GH_PACKAGES_TOKEN` is a classic PAT scoped to the whole org's package
  registry, not per-repo — rotation/revocation affects every consumer at
  once; acceptable at this org size, revisit if that changes.
