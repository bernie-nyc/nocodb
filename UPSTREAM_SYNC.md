# Upstream Sync Process

This document describes how to pull a new NocoDB release into this fork
and publish an updated Docker image to GHCR.

## Pinned version

| Field         | Value                                      |
|---------------|--------------------------------------------|
| Release tag   | 2026.06.0                                  |
| Commit SHA    | 57e2383e3293f22623bcdf598c4ac3385300b640   |
| GHCR image    | ghcr.io/bernie-nyc/nocodb:2026.06.0        |
| Mirrored from | docker.io/nocodb/nocodb:2026.06.0          |

---

## When to sync

Sync is a deliberate, manual decision. Do not set up automatic pulls.
Before syncing, check the upstream release notes for:

  - Breaking changes to the database schema (NocoDB manages its own
    internal metadata tables; a schema migration failure can corrupt
    the NocoDB state even if the Postgres data tables are fine)
  - License changes (NocoDB uses Fair Code; watch for terms that affect
    self-hosted internal use)
  - Changes to webhook payload format (the mnemosyne audit webhook
    in apps/intake/views/ parses the NocoDB event payload)

---

## Sync steps

1. Check the upstream releases page for the version you want to pull:
   https://github.com/nocodb/nocodb/releases

2. Fetch the new tag from upstream into your local clone of this fork:

       git remote add upstream https://github.com/nocodb/nocodb 2>/dev/null || true
       git fetch upstream --tags

3. Reset the stable branch to the new release tag:

       git checkout stable
       git reset --hard <new-release-tag>

   Note: this is a hard reset, which rewrites history on the stable branch.
   The only commits that belong on stable beyond the NocoDB source are the
   files added by this repo (this file, .github/workflows/mirror-image.yml).
   After the reset, re-apply those files if they were lost:

       git checkout HEAD~1 -- UPSTREAM_SYNC.md .github/workflows/mirror-image.yml
       git commit -m "chore: restore repo files after reset to <new-tag>"

4. Update PINNED_VERSION in .github/workflows/mirror-image.yml to the new
   release tag string (e.g. "2026.07.0"). Commit the change.

5. Update the "Pinned version" table in this file to the new tag and commit SHA.

6. Push the stable branch to origin:

       git push origin stable --force-with-lease

   The push triggers the mirror-image.yml workflow automatically. Watch the
   Actions tab to confirm the new image is published to GHCR.

7. Update docs/nocodb_setup.md in the mnemosyne repo with the new image tag,
   then update the docker-compose or systemd unit on the droplet.

---

## From-source build (fallback)

The mirror-image.yml workflow pulls the pre-built upstream Docker Hub image.
If Docker Hub ever becomes unavailable, a from-source build is possible using
the NocoDB source in this fork. The NocoDB build process as of 2026.06.0:

  Requirements: Node.js 18+, pnpm 9+, Docker with BuildKit

  Build steps:
    cd packages/nocodb
    pnpm install
    pnpm run build
    docker build -t ghcr.io/bernie-nyc/nocodb:<tag> .

  The exact build steps may change across versions. Check packages/nocodb/
  and the upstream CI workflows (.github/workflows/) for the current process
  before attempting a from-source build on a newer version.

---

## Why this process is manual

Automatic syncs can silently introduce breaking changes -- schema migrations,
API contract changes, or webhook format changes -- that break the mnemosyne
application without warning. Every sync should be a conscious decision with
a changelog review. The push to stable and the resulting image build are the
only automated steps; everything before that is deliberate.