# Evidence Report Methodology

## Standards (defaults, overridable by the user)
- Period: 30 days, ending now (UTC).
- Minimum success rate: 95 percent.
- RPO tolerance: 1.5 times the policy interval.
These are a baseline, not a regulatory requirement.

## Verdict rules (every active protection group, paused ones included)
Let `freqH` be the policy backup interval in hours (unknown or ProtectOnce = 24). `staleH = max(48, 2 x freqH)`.
- **Evidenced**: at least one restorable recovery point in the period, success rate at or above the minimum, last success within `staleH`, and longest gap between successes at or below 1.5 x `freqH`.
- **Partial**: has a restorable recovery point and a last success within `staleH`, but the success rate or the gap rule is not met.
- **Gap**: no restorable recovery point in the period, or the last success is older than `staleH`, or the group never ran.
A paused group with a fresh success can be Evidenced. Write "Paused" in the reason.

## Per-group measures
- `runs`: finished runs only (Succeeded, SucceededWithWarning, Failed). Log runs (`kLog`) are ignored.
- `ok`: Succeeded plus SucceededWithWarning. `failed`: Failed. Success rate = ok / runs x 100, "n/a" when runs is 0.
- Last success: hours since the end of the latest ok run.
- Restorable recovery points: ok runs whose local snapshot has not been deleted (the run's `isLocalSnapshotsDeleted` is not true, which needs per-run object detail), plus ok runs whose local snapshot is gone but have a successful, unexpired replication or archival copy. A snapshot only counts if its expiry is later than now.
- Newest point age in hours. Oldest point age in days.
- Longest gap: the longest time in hours between consecutive ok runs, also counting from the start of the data read to the first ok run (only if the group had earlier runs) and from the last ok run to now.
- Offsite: ok when the latest run has a successful replication or archival result, failed when the latest attempt failed, none when there is no such target or the copy is only Accepted or Skipped. A policy whose primary backup target is Archival counts as offsite.
- Immutable: true when a run has DataLock constraints or the policy has DataLock.
- Cloud-archive-direct groups have no local backup info. Use the archival result status and times instead, and count a copy as a recovery point only if it has not expired.

## Reasons
Write a short reason for every group that is not Evidenced, for example "Success rate 61% (limit 95%)", "No successful run in 52 h", "All 30 runs failed, no restorable point", "Only archive copy has expired", "Paused for 34 d, last success 810 h ago".

## Evidence ID
`RER-` + UTC date `YYYYMMDD` + `-` + `HHMM` + `Z`.

## Capping
A group with more than 100 runs in the period is paged backwards, at most 3 extra calls. If still capped, say so and start the gap check at the oldest run read.

## Limitations statement (always include)
This report is derived from Helios backup run and snapshot metadata through a read-only connector. It shows that backups ran and succeeded, that restorable recovery points exist, how the schedule was met, and whether an offsite copy and immutability are configured. It does not show that a restore was performed or tested, and it is not a statement of regulatory compliance. Pair it with restore-test records when an auditor asks for them.
