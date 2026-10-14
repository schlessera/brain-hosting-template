# Decision — a thin source hosting template

The [maintainer ruling of 2026-09-28](https://github.com/schlessera/brain-kit/issues/70#issuecomment-5869906635)
selects a source template that builds its own container. Application behavior
arrives through published brain-kit packages. The host and content repositories
remain separate and meet through a runtime mount. This record defines the
composition; it does not certify extraction, a built image or availability.

## Inventory and ownership

These paths are the extraction contract. A generated repository contains the
same tracked files as the template; GitHub template generation is not a filter.
No maintainer-only AGENTS.md, CLAUDE.md, skills or tracker policy belongs here.

| Files | Maintained by | Customization and update boundary |
| --- | --- | --- |
| `package.json`, `bun.lock`, `tsconfig.json` | Template; generated-host owner reviews updates | Published package dependencies and build scripts only. Host needs no release version. Review ranges, actual resolutions and SDK overrides together. |
| `bin/server.ts` | Template | Await package `createApp`, register package backends/renderer, serve static output with Bun, wire shutdown. No routes, permission rules, storage algorithms or copied app behavior. |
| `src/client.tsx`, `src/styles.css`, `index.html` | Template | Mount package UI and import its shipped styles; identity and branding come from user files. No copied components or state machines. |
| `src/service-worker.ts`, `public/manifest.webmanifest`, `public/offline.html` | Template | Use public SDK worker/cached-shell APIs; host owns asset URLs, manifest declarations and build integration. #10 owns track share declarations; #11 owns cached/offline shell integration. |
| `Dockerfile`, `.dockerignore`, `compose.yaml`, `bin/entrypoint.sh` | Template | Frozen image install/build, mounts and process wiring. Do not download dependencies or update runtimes at boot. Container policy is qualified, never inferred from this inventory. |
| Native manager service/configuration and fixed store mounts | Template; capacity, reserve and device chosen by the generated-host owner | Compose the released `brain-native-store` service outside app/worker authority. Keep its fixed root-owned configuration outside the brain, share the required PID namespace and managed paths, and stop the app before the manager. No template storage engine, automatic formatting or growth. |
| `config/Caddyfile` | Template | One documented HTTPS reverse-proxy path. Domain supplied by user; no hardcoded installation identity. Keep backend ports private. |
| `.env.example`, `config/host-requirements.ts` | Template | Generic placeholders and checked-in host minima; descriptors and package enforcement own environment semantics. No credential values or default public unauthenticated mode. |
| `.env`, `config/local.env`, `branding/` | Generated-host user | Secrets and site identity. Real environment files are ignored and never copied into image layers; branding assets are explicitly user-owned. Preserve these during upgrades. |
| `.gitignore`, `README.md`, `docs/setup.md`, `docs/upgrading.md`, `docs/troubleshooting.md`, `LICENSE` | Template | Self-hoster instructions and legal text only; no contributor tracker/agent policy. License must be supplied before public distribution. |
| `scripts/build.ts`, `scripts/check-host.ts`, `scripts/check-shell.ts`, `scripts/check-leakage.ts`, `scripts/check-pr.ts`, `tests/`, `bunfig.toml`, `.github/workflows/ci.yml` | Template | Build/verification glue, own-importer checks, frozen installs, offline tests and whole-tree leakage checks. No alternative application enforcement. CI uses GitHub Actions. |

The brain's Markdown, config, own package manifest/lockfile and disposable
`brain.db` belong in the separately generated content repository, never here.
Server authority/state and agent state use separate persisted locations defined
by the qualified container. Volumes, image output, generated client assets and
local receipts are runtime/build artifacts, not new source-of-truth files.
Do not copy the historical privilege record's paths into a Dockerfile without
checking the current package boundary and real hardening evidence.

## Setup and compatibility

The supported first route is a qualifying Linux container host behind HTTPS.
WSL2 is a supported execution row only after the same tests pass; browser
clients may use other operating systems. No bare-Bun deployment path or
prebuilt-image publication pipeline is selected.

Generate a private brain and a separate private host. Before building, verify
that each chosen package version is actually published and contains the APIs
used by that host. Install each repository from its own frozen lockfile,
retaining SDK optional platform packages for its own architecture/libc.
Compile the host against installed ordinary exports, with no source-condition
forcing, workspace links or private checkout dependencies.

The server composition follows the public
[server entry point](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/index.ts#L1-L23)
and [package example](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/README.md).
These source links explain the intended composition; they do not assert that
all current-main APIs are on the registry. #7 establishes declarations and
own-importer build proof against the selected containing release. The content
CLI's minimum, loaded backend SDK and executed binary are separate identities.
Both content and host installs must resolve the bundled Claude runtime; #1
owns the container proof and its content dependency is brain-template#1.

Populate ignored configuration/secrets and user branding. Consume the package
[environment descriptors](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/config/env.ts)
and [generated reference](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/README.md#environment).
Reference these instead of maintaining a second full environment table. Only
setup-relevant examples belong in `.env.example`. Keep auth enabled, verify
proxy trust/exposure, then perform the documented startup refusal and login
checks before starting an agent. Login/token minting remains maintainer-reserved
where #1 says so; keyless proof can precede it.

Published 0.43.0 changes the hosting prerequisites: `createApp` requires the
managed native Btrfs service. Follow the
[hosting requirements and deployment sequence](../hosting-requirements.md)
for the released launcher/configuration, private-path boundaries and stopped
import. The finite physical store, 10 GiB logical retained-history cap and
RAM/tmpfs worker budgets are distinct. Total capacity and recovery reserve are
explicit operator choices. #20 owns generated native-store wiring and its
clean-room proof; this specification does not deliver those files or certify
the generated container.

## Two upgrade procedures

**Dependency upgrade:** back up content and persisted authority/auth state;
record current image digest, manifests, lockfiles and configuration revisions.
Update compatible host package ranges and separately review the content CLI
pin. Refresh both lockfiles in their respective repos. Inspect actual loaded
packages, SDK platform binaries and package/host minima, then install/build
frozen and repeat required keyless container/browser proof. Compatibility is
not evidence of measurement; compare reported versions with the recorded
package measurement tuple. Redeploy only after the applicable checks pass.
No `claude update`, entrypoint updater or automatic runtime install is allowed.

**Copied-file upgrade:** compare the generated host's originating template
commit with the desired reviewed template commit. Apply a reviewed diff to
Dockerfile, Compose, proxy, entrypoints, worker and build/test glue. A package
bump does not update any of these files. Preserve ignored secrets, branding
and explicit local configuration; resolve overlap as a review, never a reset.
Rebuild frozen and repeat proof for changed capabilities. Record the new
template commit in the host's own upgrade commit, not a status field here.

**Rollback:** retain the previous image digest and configuration/file revisions.
Restore deployment glue and configuration together; independently decide
whether to restore content manifest/lockfile. A newer compatible content CLI
may remain pinned; an image rollback does not change it. Restore backed-up
persisted state only if its compatibility requires it, preserving Markdown
and treating `brain.db` as regenerable. Test a disposable upgrade/second boot/
rollback before recommending ownership or volume changes. Do not borrow the
historical record's unverified migration commands as a tested procedure.

## Hardening and clean-room evidence

The epic's hardening-before-extraction prerequisite remains binding. Before
extracting deployment files, obtain a dated, sanitized receipt for an actually
running non-root/container agent separation: image/source identity, ordinary
server/worker identities, immutable executable ownership, authority-state
inaccessibility, cancellation and child cleanup, and upgrade/restart evidence.
A historical design or package unit test is insufficient. Public evidence
uses generic paths/ids and exact component versions; no installation host,
domain, data, private repository link or credential may enter this repository.
If the receipt cannot be supplied or verified, extraction remains blocked.

Before the first extracted commit, enable the whole-tree leakage gate,
including docs/assets/fixtures, and test that an injected prohibited sentinel
is rejected for the intended assertion. Keep pattern sources shared with the
brain-kit gate through a reviewed copy with source identity; no directory
exemptions for deployment-looking files. Gate proposed issue/PR text too.

Use only generated public project inputs and the Odysseus fixture world pinned
to 2026-07-12. A clean-room operator must be able to follow README/setup without
access to the source installation. Build from frozen published dependencies,
record the kernel/architecture/Bun/bubblewrap/browser/SDK/runtime/filesystem
and image tuple, and check positive and denied-capability controls as the
ordinary runtime user. Host-side byte/directory observations prove unchanged
brain content. #13 owns the actual worker probe, scratch/descriptor/hardlink
and refusal proof; #5 owns real PDF/glyph/cache proof; #7 owns import and
constraint proof; #10 owns actual multipart share/storage proof. These proofs
must run without inference calls or credentials and record failing mutations
and restored passes. #1's successful subscription turn is a separate final
reserved verification and may not be replaced by keyless fixtures.

The extraction task supplies a shell guard permitting only package imports,
composition, lifecycle/static-asset wiring and build/test glue in executable
host files. Review the guard's allowlist and every executable source; prove
that a planted application route/permission implementation fails it. Runtime
behavior found during extraction gets an owning brain-kit issue, not a copied
implementation. The clean-room task checks generated file lists, secret
exclusion, frozen reproducibility, setup, upgrade, second boot and rollback,
then separately records any remaining reserved live check. Availability docs
change only after the generated host works and is available to strangers.

## Dependency order

[#15](https://github.com/schlessera/brain-hosting-template/issues/15) owns
extraction after verified hardening and published containing APIs. Existing
#1/#5/#7/#10/#13 deliver runtime, rendering, requirements, sharing and worker
qualification against that shell; #11 owns cached/offline assets.
[#16](https://github.com/schlessera/brain-hosting-template/issues/16) owns the
combined generated-repository walkthrough and upgrade/rollback evidence after
those scoped prerequisites. The reserved live check and usable availability
remain part of the epic's definition of done. These dependencies define the
proof order; they are not a release schedule or a claim of completed work.

## Alternatives

A source tree generated/published from brain-kit adds release machinery and a
second hosting source location; direct maintenance here follows the existing
repository boundary. A prebuilt image hides customization/build ownership and
was not selected. A bare-Bun path adds another unqualified platform. Merging
host files into the content template couples infrastructure updates to notes.
A user-owned whole-tree manual merge offers no clear protection for secrets
and branding; the selected inventory makes those boundaries explicit while
retaining review of copied deployment glue.
