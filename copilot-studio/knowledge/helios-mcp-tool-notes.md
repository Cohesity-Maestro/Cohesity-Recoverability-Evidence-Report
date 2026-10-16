# Helios MCP Tool Notes

Reference for the agent. Use these rules whenever you call the Helios MCP tools. All tools are read-only.

## General
- Time filters are epoch microseconds in UTC (seconds x 1,000,000). Always state times in UTC.
- Call `dataprotect-get-clusters` first. Only clusters with `isConnectedToHelios: true` can be queried, using their `clusterId` as `accessClusterId`. A cluster identifier for search calls is `<clusterId>:<incarnationId>`.
- If `dataprotect-get-clusters` itself returns an error twice, stop and tell the user: "Unable to connect to Helios." Do not answer from memory or invent numbers. A cluster with `isConnectedToHelios: false` is not an error. List it as "not assessed (not connected)" and continue.
- If a call for one cluster times out, retry once, then retry with a smaller page (`maxResultCount: 25`) and without `includeLastRunInfo`. If it still fails, say that cluster could not be assessed (timeout) and carry on with the others. A single call can time out after about 2 minutes on large clusters, so process one cluster at a time, smallest first, and never wait for all clusters before answering.
- Never invent numbers. If data is missing, write "not available" and say why.

- Scope every call. Pass `accessClusterId` of one connected cluster in every call except `dataprotect-get-clusters` and the reporting tools. A call without it asks all clusters at once and times out after about 2 minutes. Start with `maxResultCount: 25` and add `includeLastRunInfo` only on a second, smaller call if the first succeeds.

## Tools used
| Tool | Use |
|---|---|
| `dataprotect-get-clusters` | Registered clusters, names, connection state |
| `dataprotect-get-protection-groups` | Groups per cluster. Pass `isActive: true`, `isDeleted: false`, `useCachedData: true`, `pruneSourceIds: true`, `pruneExcludedSourceIds: true`, and `includeLastRunInfo: true` when you need last-run status. Page with `paginationCookie`. |
| `dataprotect-get-protection-group-runs` | Runs per group. `id` is required. Pass `numRuns: 100`, `useCachedData: true`, and `includeObjectDetails: true` only when you need per-run snapshot flags. |
| `dataprotect-get-protection-policies` | Schedule, retention, replication and archival targets, DataLock |
| `dataprotect-get-alerts` | Alerts, for example critical and open |
| `dataprotect-search-objects` | Protected and unprotected object counts (`count` is returned even with `count: 1`) |
| `dataprotect-get-object-snapshots` | Snapshots for an object, to compare with policy retention |
| `dataprotect-get-source-registrations` | Source health. De-duplicate by `id`. |
| `reporting-get-report-preview` | Helios reports, for example `storage-consumption-group` |

## Run handling
- Use `runs.length`, never `totalRuns`.
- Count only finished runs: Succeeded, SucceededWithWarning, Failed. Ignore Running, Canceled, Accepted, Skipped, Missed.
- Ignore log runs (`runType` = `kLog`). Database groups produce hundreds of them. If the tool accepts a run-type filter, pass `["kIncremental","kFull"]` for database groups only.
- If 100 runs come back, request the remainder with `endTimeUsecs` set to the earliest returned `startTimeUsecs` minus 1.
- Paused groups produce no new runs but may have runs earlier in the period.

## Retention and frequency
- Retention comes back in mixed units whatever `durationUnit` says. Normalise to days: Weeks x 7, Months x 30, Years x 365. Use the longest of base retention, `extendedRetention` and any replication or archival target retention.
- Convert frequency to hours: Minutes / 60, Hours x 1, Days x 24, Weeks x 168, Months x 720. A ProtectOnce schedule has no frequency.
- Offsite copy means a replication target, an archival or cloud target, or a primary backup target of type Archival.
- DataLock means base retention has a `dataLockConfig`.

## Privacy
- Show cluster, group, policy and environment names. Do not show credentials, tenant GUIDs, email addresses or internal ids. Strip email addresses and GUIDs from error messages.
