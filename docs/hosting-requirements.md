# Hosting requirements and deployment — brain-kit 0.43.0

This guide covers an existing host adopting the published 0.43.0 packages.
**This repository is still a placeholder:** it supplies no runnable Dockerfile,
Compose file or application entrypoint. The
[generated-host specification](decisions/generated-host.md) defines their
ownership; [extraction](https://github.com/schlessera/brain-hosting-template/issues/15),
[native-store wiring](https://github.com/schlessera/brain-hosting-template/issues/20)
and [container qualification](https://github.com/schlessera/brain-hosting-template/issues/13)
remain separate deliverables. A package fixture pass does not qualify an
unbuilt template or a different host image.

0.43.0 requires native storage at hosted app startup, including hosts using
restricted workers. Unsupported storage refuses with setup guidance. There is
no copied-input fallback. The binding public contract is the
[native storage decision](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/docs/decisions/native-brain-storage.md)
and [released package API](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/docs/integration-contract/package-api.md#native-hosted-storage-breaking-host-requirement-1790).

## 0.43.1 Pi patch adoption

The published 0.43.1 patch repairs Pi SDK startup state routing while preserving
the existing native store, worker protections and persisted authentication.
Verify the matching 0.43.1 package identities against the registry before
adopting them. Follow the existing
[dependency-upgrade procedure](decisions/generated-host.md#two-upgrade-procedures):
update the host's matching package versions and lockfile, review the loaded
SDK/runtime identities, then perform a frozen rebuild and qualification.
Keep the separately installed content CLI compatible. An existing qualified
0.43.0 store does not need reprovisioning or a history reset for this patch.

If Pi uses global npm packages, prepare the operator-configured packages in
the selected Pi agent directory, outside the brain, before starting turns.
A settings entry alone does not prove an installed, matching package. Ordinary
workers reuse that directory's prepared `npm/node_modules` tree read-only;
SDK settings, models and login writes use bounded private per-turn state.
Keep the prepared package tree stable while turns initialize. Missing or
mismatched configured packages report a setup error, rather than installing
them during the turn ([backend setup requirement](https://github.com/schlessera/brain-kit/blob/f691c5b62a0ffcca67bedfee0be6557631d0930d/packages/ui-backend-pi/README.md#L295-L301)).
This does not require new plugins or an agent-directory override. Prepare
trusted installed resources through the host's reviewed setup procedure;
do not run repository configuration or package installation from root boot.

## Required topology

Use a qualified Linux x86-64 container host. The native manager runs as root in
the initial user namespace; trusted OverlayFS setup checks that identity.
Use **Bun 1.4.2**, the runtime pinned by the release's
[public native CI](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/.github/workflows/ci.yml),
for this qualified route. A different Bun version needs its own native
readiness/runtime proof; satisfying a package engine range alone does not
establish qualification or a new universal minimum.
Rootless or user-remapped manager containers do not satisfy it. The manager's
mount/device authority stays outside the app and agent. Run the app as a
dedicated non-root user, with no administrative capabilities granted to its
workers. This follows the template's existing
[extraction hardening](https://github.com/schlessera/brain-hosting-template/issues/15)
and [worker qualification](https://github.com/schlessera/brain-hosting-template/issues/13)
requirements. The package also supports qualified UID 0 contexts with a
[standing authority warning](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/docs/hosting/agent-workers.md#running-the-server-as-root);
that warning does not establish this template's non-root separation. Do not
solve a worker refusal by giving the app `CAP_SYS_ADMIN` or making the whole
app privileged.

The manager and app must see **the same PID namespace and the same absolute
managed paths**. The app enters an owned staging keeper's user/mount namespace;
an unrelated container PID namespace cannot resolve that keeper safely.
Sharing a socket volume alone is insufficient. A qualified supervisor can run
the separate manager and unprivileged app in one container/PID namespace;
separate services require an explicitly shared PID namespace and consistent
mounts. Neither topology is certified merely by drawing it in Compose.

| Component | Required access and boundary |
| --- | --- |
| Root-owned manager | One fixed store, actual block device and root-owned configuration. It performs only released checkpoint/staging operations; it accepts no arbitrary command or caller-supplied path. |
| App | Access to the live brain, sealed inputs/staging and manager Unix socket, plus its own persisted authority/history state outside the brain. Configure the manager's fixed `appUid`/`appGid` for the actual application identity. |
| Worker | The package's admitted input/output view and existing masks. No manager socket, private native state, backing file or block device. Read-only mounts alone do not prevent Unix-socket connections. |

`createApp` verifies the live brain's canonical path and native identity and
uses the manager's private paths as worker masks. Keep the socket in a dedicated
protected parent, not inside the brain. Preserve all existing worker/secret
masks; do not replace package masking with an ad hoc read-only bind.

The manager needs trusted root-owned `/usr/bin/btrfs`, `/usr/bin/python3` and
`/usr/bin/fallocate`; staging also uses `/usr/bin/mount`, `/usr/bin/umount`,
`/usr/bin/unshare` and `/usr/bin/setpriv`. The app/worker image needs the released
bubblewrap and util-linux launcher tools, including `nsenter` and `prlimit`,
and the existing FIFO/network tooling. Frozen image builds install these;
boot must not install tools or discover mutable executables from the brain.

Preserve the [released Docker worker requirements](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/docs/hosting/agent-workers.md#running-the-server-in-a-docker-container):
permitted user/PID/mount/IPC/UTS namespaces, the specified namespace seccomp
calls (including `setns`), fresh `/proc`, and qualified LSM rules. On AppArmor
hosts, maintenance's paired namespace restrictions need the permitted
namespace-local `/proc/sys/user/max_user_namespaces` setup as well as mounts
and `pivot_root`. Preserve the other proc/sysctl denies and the worker's
capability drop. Use the actual ordinary **and maintenance/governed** probes;
an unconfined or unenforced local LSM run is not enforced-profile proof.

## Three different capacity domains

| Domain | What it bounds |
| --- | --- |
| Fixed physical managed store | Live brain, temporary native checkpoints, manager persistence/reserve and filesystem overhead on that device. Anything else placed on the same device also consumes it. The operator explicitly chooses its finite capacity and recovery reserve. |
| 10 GiB logical retained history | Distinct retained changed-file objects, journal metadata and object-write reservations. Shared physical extents do not reduce these logical charges. Keep history/authority storage outside the brain and plan its actual disk placement separately. |
| Worker RAM/tmpfs budgets | Runtime state, turn temp and private staged outputs, each with its existing per-worker allowance and aggregate admission budget. These are separate from disk capacity; admission accounting does not reserve host RAM. |

History is not a monthly allowance or a collection of full-brain snapshots
accumulating each month. Short-lived checkpoints survive only while execution,
observation or recovery owns them; changed eligible versions enter retained
history. Do not interpret the 10 GiB history cap as the total live-brain volume
size, add it automatically to the volume, or assume checkpoints have no cost.
There is no automatic volume growth. No universal volume or reserve size is
provided: choose both from your data, write/concurrency workload, filesystem
overhead and measured recovery needs.

The hosted output-staging default is 256 MiB per operation. Its aggregate
default is that allowance multiplied by `MAX_CONCURRENT_SESSIONS`, unless
`BRAIN_UI_OUTPUT_STAGING_BUDGET_BYTES` is explicit
([released descriptors](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/config/env.ts#L363-L378)).
Inputs now use native checkpoints; this allowance bounds writable tmpfs
outputs, not complete input copies. Match the manager's staging limits to the
app's actual resolved output limits. See the
[worker budgets](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/docs/hosting/agent-workers.md).

## Before changing an existing installation

1. Back up Markdown and its Git/config/package files, plus persisted UI
   authority/auth/history state, native lease ledgers and manager state.
   Record image digests, installed package/SDK/runtime identities, lockfiles,
   configuration and mount/ownership layout. Keep a recoverable backup outside
   the device being changed.
2. Stop the app, cron, autonomous jobs and every external brain writer. Drain
   owned workers and observations before stopping the manager. Verify process
   and mount cleanup; do not import or alter live-subvolume identity while an
   app or reader still owns a checkpoint.
3. Provision/import as an explicit administrator operation. The manager does
   not format a disk, create the live brain, resize a store or migrate notes.
   Never point a formatting command at existing data based on this guide.
   Copy a stopped brain as data, preserving its file/Git metadata and reviewed
   ownership; do not execute its config, hooks or package scripts as host root.
4. Before starting the manager, app or cron, and **before any checkpoint**,
   prepare the imported brain's protected Git directories as described below.

**Prepare protected Git directories before the first checkpoint.**

While all writers remain stopped, inspect the canonical live brain and its
`.git` directory without following symlinks. Require real directories; stop
if `.git` is a symlink, a Git worktree pointer file or another unexpected type.
Inspect `.git/hooks`, `.git/info` and `.git/modules` the same way:

- If a path is already a real directory, preserve its contents, ownership and
  permissions. Do not empty it or recursively change ownership.
- If a path is absent, create only that empty directory with
  `mkdir --mode=0755 -- PATH`, using the reviewed Git metadata owner and group.
  Verify its actual type, ownership and 0755 mode afterward; an active umask
  can reduce the requested mode. Do not use `mkdir -p` to hide parent mistakes.
- If a path is a symlink or any non-directory type, stop and investigate rather
  than replacing it. This step does not execute Git, hooks, config or packages.

Published 0.43.0 can take a
[standalone maintenance checkpoint](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/brain/host-boundary.ts#L438-L449)
before
[`resolveWorkerProtection` creates absent protected directories](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-sdk/src/server/worker-protection.ts#L39-L75).
An empty directory then appearing in the worker output can be refused as a
protected-input change by the
[native publisher](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-sdk/src/server/worker-native-output.ts#L148-L160).
Prepare all three before the first snapshot; do not rely on a status request
to prepare them. This is an initial-store workaround, not a runtime fix or a
change to protected-path enforcement.

For the selected store, provide one **read-write top-level Btrfs mount** on one
fixed block device, with the live subvolume at `STORE/brain`. The store and its
ancestors are canonical root-owned directories not writable by group/others;
the live brain has the application's required content ownership. Use
`nosuid,nodev,nodiscard`, disable Btrfs qgroups, and allow no child subvolume or
nested mount inside the brain. A directory on another filesystem, a store mounted from a subvolume rather than its top-level tree, or a read-only
live root is unsupported.

Check the actual device length with `blockdev --getsize64 DEVICE`, mount source
and options with `findmnt`, and filesystem/subvolume identities with
`btrfs filesystem show --raw STORE` and `btrfs subvolume show STORE/brain`.
The manager requires `capacityBytes` to equal the actual sole device capacity.
Record the filesystem UUID and the live subvolume UUID; changing the latter is
a new brain identity, not a harmless path rename.

A loop device is supported only with a fixed, root-owned, fully allocated
regular backing file whose length equals capacity, with one hardlink and
protected ancestors. The service verifies its kernel backing device/inode,
allocation, zero offset and no implicit size limit. A sparse image's apparent
length is insufficient. **Disable discard explicitly**: discard can punch
holes in otherwise preallocated backing. `nodiscard` does not prevent an
explicit `fstrim`: exclude this fixed loop-backed store from scheduled and
manual trim as well. Do not resize/autogrow either the device or its backing
after qualification.

Keep the canonical backing file in a dedicated protected root-owned parent
outside the brain; hide that parent and every other backing-file alias from
workers. Verify with an actual worker that neither backing bytes nor the
device/socket/private native paths are readable or reachable. A read-only
regular-file mask alone does not establish backing-file secrecy; the
[known visibility gap](https://github.com/schlessera/brain-kit/issues/1834)
requires careful directory/alias protection and real denial proof.

## Configure and start the released services

Paths such as `/srv/brain-store`, `/run/brain-native` and `/var/lib/brain-host`
below are generic examples, not default locations or capacity policy. Choose
consistent canonical paths for both services. Build the host and content
installs separately from their own frozen lockfiles; use ordinary published
package exports. Include the published `@schlessera/brain-ui-sdk@0.43.0` in the
host install that supplies `brain-native-store`. Preserve the qualified SDK
and its architecture/libc binary dependencies rather than updating at boot.

The [released launcher](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-sdk/src/bin/brain-native-store.ts)
takes one canonical absolute root-owned JSON file, at most 16 KiB. Its file and
ancestors must not be group/other-writable. Keep the file outside the brain and
image secrets outside tracked source. Populate these actual
[configuration fields](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-sdk/src/server/native-brain-manager.ts#L39-L56):

| JSON field | Value to supply |
| --- | --- |
| `storePath` | Canonical top-level store mount, for example `/srv/brain-store`; its live root is always `storePath/brain`. |
| `devicePath` | The actual sole fixed block device, not the backing-file pathname. |
| `fsid` | Lowercase hyphenated Btrfs filesystem UUID. |
| `brainUuid` | Live subvolume UUID as 32 lowercase hexadecimal characters, without hyphens. |
| `socketPath` | Absolute Unix socket, for example `/run/brain-native/manager.sock`, outside the brain with protected root-owned ancestors. |
| `appUid`, `appGid` | Actual fixed application user/group IDs. The socket is root-owned and group-accessible; this does not grant workers manager access. |
| `capacityBytes` | Operator-chosen positive finite integer equal to the actual device's byte length. |
| `recoveryReserveBytes` | Operator-chosen positive finite multiple of 4096, smaller than capacity. The manager verifies actual emergency-file allocation; its size alone is not a metadata recovery guarantee. |
| `stageAllowanceBytes`, `stageBudgetBytes` | Positive page-aligned integers matching the app's output-staging allowance and aggregate budget; allowance must not exceed budget. Supply these with `appUid` for hosted maintenance/staging. |
| `operationTimeoutMs`, `maxDeltaBytes` | Optional finite positive transport bounds; defaults 30000 ms and 67108864 bytes. Neither is a storage/history quota. |

Start the manager from its immutable host install, as a separately supervised
trusted service, never from the content repository:

```sh
/opt/brain-host/node_modules/.bin/brain-native-store /etc/brain-native-store/config.json
```

This command runs as the trusted manager identity, not the app user. Its
`Native brain storage is ready.` message qualifies the store primitives; it
does not establish the application's complete worker posture.

Set the app's [released environment descriptors](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/README.md#environment)
in ignored host configuration:

```sh
BRAIN_PATH=/srv/brain-store/brain
DB_PATH=/var/lib/brain-host/brain-ui.db
BRAIN_UI_NATIVE_BRAIN_SOCKET=/run/brain-native/manager.sock
BRAIN_UI_NATIVE_BRAIN_LEASE_PATH=/var/lib/brain-host/native-leases.sqlite
```

The lease path is persistent and outside the brain, together with its SQLite
companions. It otherwise defaults beside `DB_PATH` and must be explicit for an
ephemeral UI DB. Keep it across restarts so dead-owner checkpoints and stages
can be reconciled. The app rejects mismatched brain paths/identities; do not
reuse a lease ledger for another brain. Preserve separate cron configuration
and the same native manager/lease topology for scheduled maintenance.

Optional `BRAIN_UI_NATIVE_BRAIN_TIMEOUT_MS` and
`BRAIN_UI_NATIVE_BRAIN_DELTA_BYTES` configure the client's transport bounds.
Align them with manager work bounds if changing them. Existing
`BRAIN_UI_OUTPUT_STAGING_BYTES` and `BRAIN_UI_OUTPUT_STAGING_BUDGET_BYTES` select
the app staging limits; these are independent of native disk capacity.
Keep auth enabled and backend ports private behind HTTPS; native storage is
not a replacement for authentication, worker isolation or resource limits.

## Qualification, shutdown and recovery

Start the manager first, then the unprivileged app using package `createApp`.
Wait for manager readiness, await app startup and open its listener before
starting separate cron module discovery. Initial app qualification and cron
discovery share maintenance reservations; starting them concurrently can leave
the initial governed posture unavailable. Manager readiness alone is not the
signal to start discovery.

For an independently supervised cron process, wait with a bounded deadline on
`http://127.0.0.1:PORT/api/health` in the app's network namespace, using its
configured port. Require HTTP
200 and JSON `status: "healthy"`; send no auth and accept no redirects. On
timeout or refusal, preserve diagnostics and leave discovery stopped. The
[released composition](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/README.md#L19-L36)
opens the listener after awaited `createApp`, which
[awaits initial boundary preparation](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/app.ts#L380-L381).
The [public health route](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/packages/ui-server/src/routes/health.ts#L13-L36)
checks app/database liveness; it does **not** prove governed availability.
Keep the separate status and execution qualification below.

Do not run repository configuration, Git hooks, a content CLI or `bun install`
from the root boot/supervisor path. Content-executing maintenance belongs in
the package's admitted workers, after native and worker qualification. A
previous image's boot-time sync script is not made safe by adding a socket.

Before enabling writers, inspect startup diagnostics and the authenticated
status/UI host requirements. Require healthy ordinary and governed/maintenance
postures, with the expected non-root application identity, selected runtime
versions and limits. Verify first and warm turn completion, an actual indexed
write, cancellation, cleanup, restart and an unavailable/mismatched-store
refusal without provider keys in a disposable Odysseus fixture. Do not accept
a strict fallback or a warning as proof that the requested governed mode ran.

The upstream [native hosted fixture](https://github.com/schlessera/brain-kit/blob/5dc2595ca07042f2e2024c95bbc361f31637559f/scripts/native-proof/README.md)
and `bun scripts/prove-native-host.ts` in that released source checkout provide
keyless package evidence with recorded environment/LSM identity. Their own
disposable image and fixture are not proof of a generated or locally changed
host image. The template's native/container tasks require that additional
clean-room proof before claiming supported deployment.

Wire termination so the app stops accepting work, cancels and **awaits** its
owned turns/journal/resource cleanup, then stops the manager. Keep the manager
available throughout that drain. Set the outer supervisor/Compose stop grace
from measured cancellation, capture and release times rather than killing the
manager first or assuming a generic short timeout suffices. Abrupt termination
can leave durable unknown outcomes; restarting must reconcile ownership, not
delete every checkpoint or clear ledgers to make startup appear clean.

An absent socket, wrong filesystem/device/UUID, discard/qgroups, unsupported
namespace/LSM topology, unallocated backing or exhausted reserve must produce
an explicit refusal. Preserve diagnostics and durable state. Reclaim only
verified eligible owned roots through the released recovery path; do not
blindly remove private directories or another live app's leases. Physical
deletion/reclamation must finish before that space is considered reusable.
Keep independent status/recovery available and increase configured transport
bounds only for a measured work-bound refusal, never as a storage workaround.

For rollback, stop/drain all writers first and retain the current backup,
lease/manager state and data. Restore the previous image, copied host files,
mount topology and configuration as a reviewed unit. Review content
manifest/lockfile compatibility separately: rolling back an image does not
change the content CLI pin. Do not format the store, change its live UUID or
discard authority/history state as part of an image rollback. Reverting to a
pre-native host requires its own tested storage/configuration migration; it
is not accomplished by removing the new environment names.
