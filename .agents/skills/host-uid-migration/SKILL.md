---
name: host-uid-migration
description: Prepare, qualify and roll out a brain-kit host's non-root or separate-worker-UID migration, including persistent state and rollback. Use for a root-worker warning or a planned host identity change.
---

Read [worker identity requirements](../../../docs/worker-identities.md) and
[hosting requirements](../../../docs/hosting-requirements.md) before changing
an image, service user or volume. This is guidance for the generated-host
operator; it grants no production access and carries no project-maintainer
tracker policy.

Establish the actual installed release and desired boundary first. A non-root
server with same-UID workers removes root authority but does not separate
workers from the server. As of 0.43.1 the separate-UID broker is not delivered;
verify the containing release and production helper proof before configuring
it. Do not suppress the root notice or install an argv-only UID wrapper as a
substitute.

Inspect the host and existing configuration before asking for facts available
there. Clarify only unresolved choices: rollout scope, host move, downtime,
backup/restore constraints and recovery cutoff. Prepare reviewable changes and
offline rehearsal while production-specific decisions are pending. Existing
task authorization applies; do not ask the operator to approve the same action
again.

Check the guide's AppArmor prerequisite when non-root namespaces fail. Prefer
an explicitly selected app-specific user-namespace permission over disabling
the host-wide restriction. Preserve existing enforcing policy; the guide's
unconfined example does not add a MAC sandbox. Verify the effective profile,
both worker paths and rendering on the real host before cutover.

Use the linked guide's ownership and proof requirements. Keep the native
manager privileged, drop identity before all content-aware execution, preserve
bounded worker storage and retain existing filesystem/subvolume identity.
Handle transcripts separately from credentials. Never run a recursive chown
of the whole store, clear a lease ledger or reinstall repository dependencies
as a root boot repair.

Inspect automatic deployment and scheduled writers before merging a migration.
Qualify the exact candidate with an executed upgrade, second boot and rollback.
Cut over only with the selected window and a recoverable stopped-state backup.
If identity, native recovery, worker cleanup or continuity checks fail, stop the
cutover and use the tested recovery procedure. Record what passed, what failed
and any remaining same-UID exposure; a process restart or health response alone
does not establish completion.
