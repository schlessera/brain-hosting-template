# Worker identities and migration requirements

Upgrading to brain-kit 0.43.1 does not change the operating-system user of an
existing container. Its workers inherit the server's outside UID. A root
server therefore still produces the standing root-authority warning. The
[released worker guide](https://github.com/schlessera/brain-kit/blob/f691c5b62a0ffcca67bedfee0be6557631d0930d/docs/hosting/agent-workers.md#running-the-server-as-root)
describes that boundary. Suppressing the warning does not change it.

There are two independently verifiable transitions:

| Transition | Server | Workers | What it establishes |
| --- | --- | --- | --- |
| Remove root execution | `brain`, UID/GID 10001 | Same outside UID 10001 | Removes UID 0 authority from app, repository execution and scheduled work. Server/worker separation still relies on package masks. |
| Separate worker identity | `brain`, UID/GID 10001 | `agent`, UID/GID 10002, shared group 10001 | Adds ownership-based denial of server-private state, after the broker and real cross-UID lifecycle pass qualification. |

These are the fixed identities selected by the
[container privilege design](decisions/container-privilege.md). Check for
collisions with host bind-mount owners before adopting them. Changing a
numeric identity later is another ownership migration.

The second transition depends on
[brain-kit #1677](https://github.com/schlessera/brain-kit/issues/1677) and
[host packaging #18](https://github.com/schlessera/brain-hosting-template/issues/18).
As of 2026-10-15 neither supplies a qualified released implementation. Do not
set the proposed `BRAIN_UI_WORKER_IDENTITY_HELPER` variable on 0.43.1 and
expect a UID switch. A plain `setpriv` wrapper cannot implement the selected
cross-UID mount access, cancellation and recovery contract.

This document defines preparation and evidence. It is not an executable or
tested ownership migration, and the template remains a placeholder.

## Settle the infrastructure before implementation

Record the chosen values and observed evidence in the generated host's private
deployment record. Keep deployment addresses, paths and credentials there.

| Requirement | Establish before cutover |
| --- | --- |
| Topology | Whether the existing host, platform and store remain, or a host move is a separate operation. Manager and app see the same PID namespace and absolute managed paths. |
| Host support | Actual kernel, architecture, Docker, LSM, seccomp, namespace support, bubblewrap and Bun versions; ordinary and maintenance worker probes on this host. See [hosting requirements](hosting-requirements.md). |
| Fixed storage | Filesystem and live-subvolume UUIDs, canonical backing device/inode, capacity, recovery reserve, free bytes and mount options. Keep an existing qualified store intact; UID migration does not format, re-import, resize or enable qgroups. |
| Resource capacity | RAM for concurrent private runtime/temp mounts and configured aggregate admission limits; disk for retained data, image build, independent cold backup and a disposable rehearsal. Do not count the recovery reserve as spare migration space. |
| Identities | Numeric app/worker users and groups, image passwd entries, supplementary groups and manager `appUid`/`appGid`. The manager retains root authority; ordinary workers retain zero capabilities. |
| Persistent paths | Every bind/named volume, source and target, owner/mode, credential store, transcript store, UI database and companions, CAS, quarantine, native lease ledger, logs and home/cache path. |
| Automation | Auto-deploy/webhook settings, scheduled platform jobs, internal cron, other repository writers, shutdown grace and a way to verify that all writers drained. |
| Recovery | Approved downtime window, maximum time before rollback, independently stored backup, verified restore destination, previous image/configuration and ownership/mode manifest. Specify how post-cutover content and rotating credentials are preserved. |
| Access | Operator authority to stop/recreate the container, alter only the selected ownership/configuration, manage service startup and inspect actual processes. Keep this authority outside workers. |

The existing root-owned native manager requires its already-qualified mount
and device privileges. Setting Docker's top-level `user: 10001` would also
drop that manager's identity. Change the app program and content-aware jobs
individually. Supervisor, system cron and syslog may remain trusted root
processes. Do not grant app/worker `CAP_SYS_ADMIN`, host PID access or an
unfiltered seccomp profile to make a failed probe pass.

## Ownership and path rules

Move executable tools out of `/root` into immutable root-owned locations
traversable by both users. A symlink that still resolves through `/root` does
not qualify. Keep the chosen, measured runtimes pinned; a UID migration is not
an opportunity to upgrade the agent CLI.

Give the app an explicit persistent home and set `HOME`, `USER`, `LOGNAME`,
`PATH` and supported backend state locations consistently in app and cron
launchers. Drop identity before cron module discovery, repository config,
Git or CLI execution. The crontab user field alone changes only the eventual
job. Preserve the app-health gate before cron discovery.

The live brain and Git metadata must support the intended writers with shared
group ownership, setgid directories, group-writable creation and the reviewed
Git configuration. Preserve intentional private file modes. The server home,
database and credentials stay private, normally app-owned directories at
`0700` and private files at `0600`. A shared group does not imply shared
access to all of `/data`.

The manager's configuration, backing storage, reserve and private intent/state
remain root-owned. Its socket needs the configured app group and narrowly
scoped parent traversal. Existing private/checkpoint/stage directories need
explicit reconciliation: changing `appGid` in the config does not change
already-created directory groups. Never recursively hand the store root or
`.brain-native` tree to the app. Preserve manager-only state denial.

The native lease SQLite file must be private and owned by the app's actual
EUID. Preserve its rows and SQLite companions; deleting the ledger or retained
checkpoints to get a clean boot loses recovery evidence. Drain old owners and
stages before changing identity. Unresolved recovery records block migration.

Inventory authentication separately from transcripts. In 0.43.1, Pi's default
server transcript directory is beneath
`$HOME/.local/state/brain-kit/pi/<brain-path-hash>/sessions`; persisting only
`.pi` does not persist those transcripts. Preserve the canonical brain path
and deliberately migrate retained transcripts. Keep Claude state, Pi login
and standalone Codex login separate; rotating grants must not be duplicated
into competing live installations. Preserve installed Pi global packages and
their selected directory; do not reinstall them during worker startup.

## Stopped migration and rollback

Prepare an immutable candidate image and a migration with a read-only plan,
explicit allowed roots and refusal on unexpected mounts, symlinks, owners or
recovery state. Rehearse both upgrade and rollback on fictional data with the
same native topology. A marker is written only after validation succeeds; a
second boot validates without repeating a blanket ownership rewrite.

Hold deployment triggers and pause every scheduled/external writer before
draining app and workers. Stop the app before the manager and verify worker
descendants, native stages and leases have drained. Killing cron alone does
not kill jobs it already launched. Take a consistent cold backup of content,
UI authority/history, credentials and transcripts, including SQLite companions,
plus a numeric ownership/mode manifest and effective deployment configuration.

Apply only the reviewed path relocation, ownership map and manager identities.
Retain the original filesystem, subvolume and backing identity. Update platform
mount destinations as well as Compose: a platform-generated container may not
consume the checked-in Compose file. Recreate from the tested image, start the
manager, verify app readiness under its new identity, then enable cron after
app health. Restore deployment triggers only after acceptance checks pass.

Rollback uses the retained image **and** matching mount targets, home paths,
manager identities and ownership. An image rollback alone leaves changed
volume metadata behind. Quiesce writers again and retain all post-cutover
content, history and current OAuth state before any state restore. Do not
overwrite later work with an earlier brain snapshot. Preserve unknown recovery
state and refuse an inconsistent rollback rather than resetting the ledger.

## Proof that makes a candidate deployable

Use the built candidate image on a disposable native store, followed by the
same checks on the selected host. Record source/image digest, package/content
CLI and runtime versions, host security options and actual process identities.

- App, ordinary worker, governed/maintenance worker, cron discovery and cron
  jobs have the intended real/effective outside UID. Inspect workers' UID maps
  and zero capability sets; `id` inside a user namespace alone is insufficient.
- The native manager remains root. Workers cannot read/connect to manager
  controls, private state, device/backing aliases or Supervisor control.
- Both backends complete first and warm turns; write/index/diff/revert and
  scratch/runtime/temp limits work. Stop/cancel and shutdown leave no worker
  tree, keeper, leaked stage or lease. A healthy HTTP endpoint is not this proof.
- Authentication survives recreation. A pre-migration session on each backend
  lists, opens and resumes with its original history. Missing transcripts are
  reported honestly. Scheduled work logs correctly; alternating writers can
  sync. Rendering and share intake work through the real application.
- Second boot preserves state and does not remigrate; deliberately unwritable
  required state refuses startup. Execute rollback and verify subsequent
  content and credential continuity. Retain meaningful failing-first/mutation
  evidence for guards changed by the migration.

For separate worker UIDs, add the full broker proof from #1677: denial of an
unmasked server-private sentinel, allowed shared writes and relays, exact
bounded mount handles before release of the start gate, private-file recovery
after all writers exit, authenticated PID-reuse-safe signalling including
nested PID descendants, STOP/CONT, supervisor/control/server loss and complete
cleanup. Failure must not silently fall back to the server UID.

Requalify Chrome separately. Non-root execution alone does not prove its
sandbox is enabled; retain the documented renderer-only accommodation until
an actual render and sandbox inspection justify changing it.
