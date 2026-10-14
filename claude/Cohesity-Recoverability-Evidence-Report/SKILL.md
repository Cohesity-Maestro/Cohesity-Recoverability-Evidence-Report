---
name: "Cohesity-Recoverability-Evidence-Report"
description: "Build the \"Cohesity Recoverability Evidence Report\" from live Helios data via the helios-mcp connector: for every protection group in scope, audit-ready evidence over a reporting period (default 30 days) of run success rate, restorable recovery points, newest/oldest recovery point, RPO gaps against the policy schedule, offsite copy status and immutability, with an Evidenced / Partial / Gap verdict, an evidence ID, a limitations statement and a CSV download. Publishes it as a live claude.ai Artifact and refreshes it every 4 hours. Use when the user asks for recoverability evidence, backup evidence for an audit (DORA, SOC 2, HIPAA, ISO 27001), proof that backups succeeded, recovery point evidence, an RPO compliance report, or to refresh/schedule that report."
---

# Cohesity Recoverability Evidence Report

Builds a single-page evidence report from live data using the **helios-mcp** connector tools, publishes it as a persistent **Artifact** (private page with a stable URL), and refreshes that same artifact **every 4 hours** via a scheduled task. The full page template is embedded at the end of this skill, so no other file is needed. Never invent numbers. If a tool fails or returns nothing, show "n/a" and say so in the final message. Never write a loose static HTML file as the deliverable. Everything is read-only.

## What this report proves, and what it does not

Helios MCP is read-only. The report is derived from Helios **run and snapshot metadata**: it shows that backups ran, succeeded and left restorable recovery points, that the schedule was met, and whether an offsite copy and immutability exist. It does **not** prove that a restore was performed or tested. The page says so in a Limitations card; keep that card and do not claim regulatory compliance. Auditors typically want evidence of backup success, recovery point availability, RPO adherence, offsite copy and immutability, plus separate restore-test records. This report covers the first group.

## What the page contains

Header with eyebrow "COHESITY RECOVERABILITY EVIDENCE", title "Recoverability Evidence Report", an **Evidence ID** (`RER-<YYYYMMDD>-<HHMM>Z`), the reporting period, the generated time and "Refreshes every 4 hours". A **cluster multi-select** recalculates, in the browser, everything below from the embedded data and is remembered in localStorage.

- Summary statement (generated from the selection) in plain sentences.
- KPI cards: Groups in scope, Evidenced %, Restorable recovery points, Evidence gaps.
- Panels: Verdict by cluster (stacked evidenced / partial / gap), Verdict by environment, Weakest groups.
- **Evidence register** (table, worst first, with a verdict filter All / Gap / Partial / Evidenced and up to 50 rows shown): group, cluster, env, runs, success %, last success, recovery points, oldest point, longest gap vs RPO, offsite copy, immutable, verdict.
- **Gaps and exceptions** with the reason for each non-evidenced group.
- **Download evidence CSV** button (all rows in the selection) using the page's `downloads` capability. Hide the button when the capability is unavailable.
- Limitations card and a footnote with the standards used.

## Standards (defaults, overridable by the user)

`standards = {"periodDays": 30, "minSuccessPct": 95, "rpoTolerance": 1.5}`. These are the skill's baseline, **not** a regulatory requirement; if the user gives their own period or thresholds, use them and say so in the scope note. Verdict rules for each active protection group:

- Let `freqH` be the policy's backup interval in hours (unknown or ProtectOnce: treat as 24). `staleH = max(48, 2 x freqH)`.
- **evidenced**: at least one restorable recovery point in the period, success rate >= `minSuccessPct`, last success within `staleH`, and longest gap between successes <= `rpoTolerance` x `freqH`.
- **partial**: has a restorable recovery point and a last success within `staleH`, but success rate or the gap rule is not met.
- **gap**: no restorable recovery point in the period, or the last success is older than `staleH`, or the group never ran.
- Paused groups are judged by the same rules (a paused group with a fresh success can be evidenced); say "paused" in `reason`.

## Data steps

Compute `now` and `start = now - periodDays` in **epoch microseconds** (`date +%s` x 1,000,000). Run independent calls in parallel. Large responses are saved to a file: process them with `python3 -I` / jq rather than reading them.

1. **Clusters**: `dataprotect-get-clusters`; keep id, incarnation id, name, `isConnectedToHelios`. Only connected clusters can be queried with `accessClusterId`. Not connected clusters are listed with `na: "not connected to Helios"`.
2. **Protection groups**: `dataprotect-get-protection-groups` per connected cluster with `accessClusterId`, `includeLastRunInfo: true`, `isActive: true`, `isDeleted: false`, `useCachedData: true`, `pruneSourceIds: true`, `pruneExcludedSourceIds: true`. A cluster call can time out: retry once, then mark that cluster `na`. Keep id, name, `environment` (strip the leading `k`), `isPaused`, `policyId`.
3. **Policies**: `dataprotect-get-protection-policies` per connected cluster (omit `includeStats`). Convert the incremental schedule to hours (Minutes/60, Hours x1, Days x24, Weeks x168, Months x720; ProtectOnce -> 24). Record whether base retention has a `dataLockConfig` (`policyLock`) and whether the policy has a replication, archival or cloud-vault target, or a primary backup target of type `Archival` (`policyOffsite`). Retention comes back in mixed units whatever `durationUnit` says.
4. **Runs in the period**: for **every** active group (paused ones too, they may have runs in the period) call `dataprotect-get-protection-group-runs` with the group's `accessClusterId`, `startTimeUsecs`, `endTimeUsecs`, `numRuns: 100`, **`includeObjectDetails: true`** (the per-run `isLocalSnapshotsDeleted` flag is absent without it; this also makes big responses, so they are saved to a file: process them with python), `useCachedData: true`. Trust the returned `runs.length`, never `totalRuns` (it ignores the run-type filter). If 100 runs came back, page backwards with `endTimeUsecs` = earliest returned `startTimeUsecs` - 1, at most 3 extra calls per group; if still capped (an hourly group has 720 runs in 30 days), set `capped: true`, say so in the scope note, and start the gap check at the oldest run actually read, not at the period start. Large or busy groups often time out: retry once, then with `numRuns: 25`, then mark the group `na` in the scope note. Batch groups; if there are more than 60, process groups with the most recent failures first and state what was not assessed.
   **Ignore log runs** (`runType` `kLog`): database groups (DB2, Oracle, SAP HANA) produce hundreds of them and they swamp the counts; use `kRegular`, `kIncremental` and `kFull` runs only and say so in the scope note.
   Per group compute from `localBackupInfo`:
   - `runs` (finished runs only: Succeeded, SucceededWithWarning, Failed; ignore Running, Canceled, Accepted, Skipped, Missed), `ok` (Succeeded + SucceededWithWarning), `failed`, `rate` = round(ok / runs x 100), null when runs is 0.
   - **Cloud-archive-direct groups** (for example a cluster that writes straight to cloud) have no `localBackupInfo`: use the run's `archivalInfo.archivalTargetResults[]` status and times in its place, and count a copy as a restorable point only when its `expiryTimeUsecs` is later than now.
   - `lastOkH`: hours since the end of the latest ok run; null when none.
   - `points`: ok runs whose local snapshot has not been deleted (`isLocalSnapshotsDeleted` not true / `hasLocalSnapshot` true) plus ok runs with a successful, unexpired archival or replication copy whose local snapshot is gone.
   - `newestH`, `oldestD`: age of the newest restorable point in hours and of the oldest in days.
   - `gapH`: longest gap in hours between consecutive ok runs, also counting the gap from the start of the data actually read to the first ok run only when the group had earlier runs, and from the last ok run to now. Null when no ok run.
   - `offsite`: from the latest run, `ok` when a replication or archival result has status `Succeeded` or `SucceededWithWarning`, `failed` when the latest attempt `Failed`, `none` when the run has no such target or the copy is only `Accepted` / `Skipped` (not yet a copy). A policy whose primary backup target is `Archival` also counts as offsite.
   - `lock`: true when any run has `dataLockConstraints` or the policy has DataLock.
5. **Verdict**: apply the standards above and write `status` and a short `reason` for every group that is not evidenced (e.g. "Success rate 61% (limit 95%)", "No successful run in 52 h", "Paused for 34 d, last success 810 h ago").
6. **Evidence ID**: `RER-` + UTC date `YYYYMMDD` + `-` + `HHMM` + `Z`, from the generated time.

## Filling the template

| Placeholder | Value |
|---|---|
| `{{GENERATED_UTC}}` | e.g. `2026-10-13 21:33 UTC` |
| `{{DATA_JSON}}` | the JSON below, valid JSON, with every `</` written as `<\/` |
| `{{SCOPE_NOTE}}` | footnote starting with `* `: standards used, clusters excluded and why (real registered count), groups capped or not assessed, any user-supplied thresholds |

`{{DATA_JSON}}` shape:

```
{"evidenceId":"RER-20261013-2133Z","periodDays":30,"startUtc":"2026-09-13 21:33 UTC","endUtc":"2026-10-13 21:33 UTC",
 "standards":{"minSuccessPct":95,"rpoTolerance":1.5},
 "clusters":[{"name":"...","connected":true,"na":"optional reason"}, ...ALL registered clusters...],
 "groups":[{"group":"...","cluster":"...","env":"VMware","freqH":24,"paused":false,
   "runs":30,"ok":29,"failed":1,"rate":97,"lastOkH":6,"points":29,"newestH":6,"oldestD":29,"gapH":49,
   "rpoMet":false,"offsite":"ok","lock":true,"status":"partial","reason":"Longest gap 49 h (limit 36 h)"}, ...ALL assessed groups...]}
```

`lastOkH`, `newestH`, `oldestD`, `gapH` and `rate` may be `null`. Numbers must tie: verdict counts for a selection sum to the groups in scope.

## Large Helios results (do this in the main session)

`dataprotect-get-protection-group-runs` with `includeObjectDetails:true` often returns more than the tool limit and the result is saved to a file, e.g. `/var/folders/.../projects/session/<sessionId>/tool-results/<file>.txt`. That host path is not readable from the Linux shell or from subagents, which is why a delegated run cannot process the data. Do the collection and the maths yourself, in the main session:

1. Call the runs tool per group (`accessClusterId` set, `startTimeUsecs`/`endTimeUsecs` for the period, `numRuns:100`, `includeObjectDetails:true`, `useCachedData:true`). Do not pass `runTypes` values other than `kIncremental`, `kFull`, `kLog`; leave it out for VM, view and NAS groups, and pass `["kIncremental","kFull"]` for database groups so `kLog` runs do not fill the 100-run limit.
2. Translate a saved path for bash: replace the prefix up to `/projects/` with `/sessions/<session>/mnt/.claude/projects/` (find it with `find /sessions -name '<file>.txt'`). Scheduled-task sessions have their own session id.
3. Process the files with python (count a restorable point only for `localSnapshotInfo.snapshotInfo.status = kSuccessful` with `expiryTimeUsecs` in the future, or a replication or archival result that succeeded and has not expired; ignore `kLog` runs) and write one row per group to a JSON file as you go, so a long run survives a context reset.
4. Results small enough to come back inline cannot be read from bash; compute their figures directly and add them to the same JSON.

## Manual refresh

The template has a **Refresh from Helios** button. A published page cannot call the Helios connector itself, so the button copies the command `Run the Cohesity-Recoverability-Evidence-Report skill to refresh this report from live Helios data`; paste it into Claude and it republishes the same artifact. Keep the button in every rebuild.

## When Helios is unavailable

Make `dataprotect-get-clusters` the first Helios call. If it fails ("Server helios-mcp unavailable", an expired token, or a timeout twice in a row), **stop collecting data and do not clear, blank or overwrite anything the page already holds**. Instead show an error on the existing page:

1. Find the live artifact (its URL is in the scheduled task prompt, or use `Artifact` `action: "list"` and match the title "Cohesity Recoverability Evidence Report" case-insensitively) and read it with `action: "read"`, `path: "index.html"`, `out_dir` set to the scratch directory. If no artifact exists yet, publish nothing and tell the user.
2. Build an HTML **fragment** from the saved page: everything from the first `<title>` up to (not including) `</body>`. Leave the data JSON, scripts and styles exactly as they are.
3. Add or replace this banner directly after `</header>` (replace the `<!--HELIOS_ERR-->` marker, or an existing `<div class="errbar"` block if one is already there; add the `.errbar` rule to the `<style>` block if the page lacks it):
   `<div class="errbar" role="alert">Unable to connect to Helios. Showing the last successful data. Last attempt <UTC time>. Reconnect the helios-mcp connector, then refresh.</div>`
4. Publish the fragment with `url` set to the artifact, `capabilities omitted` (the stored declaration is kept), so the page keeps showing its last data plus the error message.
5. Reply in one line: "Unable to connect to Helios; kept the last data and added an error banner."

A later successful run rebuilds the page from the template, which carries an empty `<!--HELIOS_ERR-->` marker, so the banner disappears by itself. Never invent or reuse old numbers as new ones: the banner is the only change on a failed run.

## Output: publish as a live Artifact

1. Write the filled template to a scratch file as an HTML **fragment** (no doctype/html/head/body) in the session's working/outputs directory (for example `<outputs>/_build/evidence.html`) or any scratch folder the Write tool can reach, never inside a git repository; keep `<title>Cohesity Recoverability Evidence Report</title>` stable. If the `artifact-design` skill is available, load it first.
2. Check whether it already exists: `Artifact` with `action: "list"`, title "Cohesity Recoverability Evidence Report".
   - **Exists** (any refresh): `action: "read"` it, then publish the new file with `url` set to it. Never create a second artifact, do not pass `icon`, omit `capabilities` so the stored declaration is kept.
   - **Does not exist**: publish with `icon: "report"`, `capabilities: {downloads: {}}` and a one-sentence `description`.
   - **Stored URL not found** (the artifact was deleted or is not visible to this account): do not stop. Publish the page as a new artifact (first-publish rules above) and tell the user in the reply: "I couldn't find the previous artifact, so I republished a new one", with the new link, and that any scheduled task prompt still pointing at the old URL needs the new one.
3. The Artifact is the deliverable. Do not also save a static HTML file.
4. Reply in 2 to 3 sentences: evidence ID, groups evidenced / partial / gap, the weakest group. State anything not assessed. On a scheduled refresh, one line.

## Refresh every 4 hours (default)

After the first publish, call `mcp__scheduled-tasks__list_scheduled_tasks` and look for a task whose title contains "Recoverability evidence". If missing, create `cohesity-recoverability-evidence-refresh` with title "Cohesity Recoverability evidence report (every 4 hours)", `cronExpression: "0 */4 * * *"` (local time), `notifyOnCompletion: false`, and this self-contained prompt: "Run the Cohesity-Recoverability-Evidence-Report skill end to end with live helios-mcp data (read-only) and republish the existing 'Cohesity Recoverability Evidence Report' artifact: find it with the Artifact list action by title (case-insensitive) or use its URL if one is given here (add it when the task is created: ARTIFACT_URL), read it, then publish the refreshed page with its url. Do not pass an icon or capabilities. Never create a second artifact. Reply in one line with the evidence ID and the evidenced / partial / gap counts. If helios-mcp is unavailable, follow the skill's \"When Helios is unavailable\" section (keep the last data and add the error banner). If the skill is not installed in Claude, say so and stop; install it from github.com/Cohesity-Maestro/Cohesity-Recoverability-Evidence-Report." If a task exists with another cadence, fix it with `update_scheduled_task`. Tell the user scheduled tasks run only while the Claude app is open, a missed run executes on next launch, and the task runs in Auto-approve mode unless they edit it. Keep the `Refreshes every 4 hours` header text in sync with the cron. Each refresh produces a new evidence ID; if the user needs a frozen report for an auditor, tell them to download the CSV from the page or ask for a one-off run.

## Rules
- Read-only against Helios: only get/list tools.
- Never state or imply that the report proves a successful restore, or that the customer is compliant with DORA, SOC 2, HIPAA or any standard.
- Do not include object-level names, credentials or internal IDs; cluster, group and environment names are fine.
- The artifact stays private; tell the user to share it from the page's Share menu.

## Page template

```html
<title>Cohesity Recoverability Evidence Report</title>
<style>
/* Layout: evidence header, cluster filter, KPI row, summary statement, three panels, evidence register, gaps, limitations. A document-like report that is also operated. */
:root{--bg:#f3f6f5;--card:#ffffff;--border:#dbe3e0;--text:#1f2a28;--muted:#5b6a67;--track:#eef2f1;--rule:#eef1f0;--hover:#eef5ef;
--green:#2e7d32;--red:#c0392b;--amber:#d4820a;--green-lt:#81b57e;--red-lt:#e59a93;--amber-lt:#e8bb74;--chip:#e8efec}
@media (prefers-color-scheme: dark){:root:not([data-theme="light"]){--bg:#121917;--card:#1b2422;--border:#2d3a37;--text:#e6eeec;--muted:#98a9a5;--track:#25312e;--rule:#26322f;--hover:#243230;
--green:#6fbf73;--red:#ef7a6c;--amber:#f0a840;--green-lt:#3d6a3f;--red-lt:#7a3c36;--amber-lt:#7a5a22;--chip:#2a3835;color-scheme:dark}}
:root[data-theme="dark"]{--bg:#121917;--card:#1b2422;--border:#2d3a37;--text:#e6eeec;--muted:#98a9a5;--track:#25312e;--rule:#26322f;--hover:#243230;
--green:#6fbf73;--red:#ef7a6c;--amber:#f0a840;--green-lt:#3d6a3f;--red-lt:#7a3c36;--amber-lt:#7a5a22;--chip:#2a3835;color-scheme:dark}
body{background:var(--bg);color:var(--text);font-family:Arial,Helvetica,sans-serif;padding-inline:16px}
*{box-sizing:border-box}
.wrap{max-width:1180px;margin:0 auto;padding-block:28px 40px}
header{display:flex;justify-content:space-between;align-items:flex-end;gap:16px;flex-wrap:wrap;padding-bottom:14px;border-bottom:3px solid var(--green);margin-bottom:16px}
.eyebrow{color:var(--green);font-size:12px;font-weight:700;letter-spacing:.12em}
h1{margin:4px 0 0;font-size:28px;text-wrap:balance}
.meta{text-align:right;font-size:12px;color:var(--muted);line-height:1.6}
.meta b{color:var(--text);font-family:"Courier New",monospace}
.filter{display:flex;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:16px}
.filter .lbl{font-size:12px;color:var(--muted);text-transform:uppercase;letter-spacing:.05em}
details.ms{position:relative}
details.ms>summary{list-style:none;cursor:pointer;background:var(--card);border:1px solid var(--border);border-radius:8px;padding:8px 12px;font-size:13px;min-width:260px;max-width:100%;display:flex;justify-content:space-between;gap:12px}
details.ms>summary::-webkit-details-marker{display:none}
details.ms>summary::after{content:"\25BE";color:var(--muted)}
details.ms>summary:focus-visible,.menu button:focus-visible,.menu input:focus-visible,.btn:focus-visible{outline:2px solid var(--green);outline-offset:2px}
.menu{position:absolute;z-index:5;top:calc(100% + 4px);left:0;width:340px;max-width:calc(100vw - 32px);max-height:340px;overflow-y:auto;background:var(--card);border:1px solid var(--border);border-radius:8px;padding:6px;box-shadow:0 6px 20px rgba(0,0,0,.18)}
.menu .acts{display:flex;gap:8px;padding:4px 6px 8px;border-bottom:1px solid var(--rule);margin-bottom:4px}
.menu button,.btn{background:none;border:1px solid var(--border);color:var(--text);border-radius:6px;padding:4px 10px;font-size:12px;cursor:pointer}
.btn{padding:6px 12px;background:var(--card)}
.menu label{display:flex;align-items:center;gap:8px;padding:6px;border-radius:6px;font-size:13px;cursor:pointer}
.menu label:hover{background:var(--hover)}
.menu .tag{margin-left:auto;font-size:11px;color:var(--muted)}
.grid{display:grid;gap:16px;margin-bottom:16px}
.g4{grid-template-columns:repeat(4,minmax(0,1fr))}.g3{grid-template-columns:repeat(3,minmax(0,1fr))}.g2{grid-template-columns:repeat(2,minmax(0,1fr))}
@media(max-width:900px){.g4,.g3,.g2{grid-template-columns:repeat(2,minmax(0,1fr))}}
@media(max-width:600px){.g4,.g3,.g2{grid-template-columns:minmax(0,1fr)}}
.card{background:var(--card);border:1px solid var(--border);border-radius:10px;padding:16px 18px;min-width:0}
.full{margin-bottom:16px}
.card h2{margin:0 0 12px;font-size:14px}
.row{display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:12px}
.row h2{margin:0}
.lbl{font-size:12px;color:var(--muted);text-transform:uppercase;letter-spacing:.05em}
.big{font-size:40px;font-weight:700;margin:6px 0 2px;font-variant-numeric:tabular-nums}
.sub{font-size:12px;color:var(--muted)}
.tool{margin-top:10px;font:11px "Courier New",monospace;color:var(--muted);border-top:1px solid var(--rule);padding-top:8px}
.red{color:var(--red)}.amber{color:var(--amber)}.ok{color:var(--green)}
.bar{display:flex;align-items:center;gap:8px;margin:8px 0;font-size:12px}
.bar .n{width:110px;flex:none;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.bar .t{flex:1;background:var(--track);border-radius:4px;height:16px;display:flex;overflow:hidden}
.bar .f{height:16px}
.bar .c{width:70px;text-align:right;font-weight:700;font-variant-numeric:tabular-nums;flex:none}
.legend{display:flex;gap:14px;font-size:11px;color:var(--muted);margin-bottom:6px;flex-wrap:wrap}
.legend i{display:inline-block;width:10px;height:10px;border-radius:2px;margin-right:4px;vertical-align:-1px}
.tw{overflow-x:auto}
table{width:100%;border-collapse:collapse;font-size:12px}
th{text-align:left;color:var(--muted);font-weight:600;padding:4px 6px;border-bottom:1px solid var(--border);white-space:nowrap}
td{padding:7px 6px;border-bottom:1px solid var(--rule);vertical-align:top}
td.num{font-variant-numeric:tabular-nums;white-space:nowrap}
td.why{color:var(--muted);overflow-wrap:anywhere;max-width:320px}
.chip{display:inline-block;border:1px solid var(--border);background:var(--chip);border-radius:999px;padding:2px 8px;font-size:11px;white-space:nowrap}
.chip.evidenced,.chip.ok{border-color:var(--green);color:var(--green)}.chip.partial{border-color:var(--amber);color:var(--amber)}.chip.gap,.chip.failed{border-color:var(--red);color:var(--red)}
.fbar{display:flex;flex-wrap:wrap;gap:8px 12px;align-items:flex-end;margin:0 0 10px}
.fbar label{font-size:11px;letter-spacing:.06em;text-transform:uppercase;color:var(--muted);display:flex;flex-direction:column;gap:4px}
.fbar select,.fbar input{background:var(--card);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:6px 8px;font:inherit;font-size:12px;min-width:0;max-width:100%}
.fbar input{width:170px}
.fbar select:focus-visible,.fbar input:focus-visible{outline:2px solid var(--green);outline-offset:2px}
.tw.sc{max-height:460px;overflow:auto}
.tw.sc thead th{position:sticky;top:0;background:var(--card);z-index:1}
.summary{font-size:14px;line-height:1.6}
p{font-size:13px;line-height:1.55;margin:0}
.foot{font-size:11px;color:var(--muted);margin-top:8px;line-height:1.6}
.empty{font-size:12px;color:var(--muted);padding:8px 0}
code{font-family:"Courier New",monospace}
.errbar{background:var(--card,#fff);color:var(--red,#c0392b);border:1px solid var(--red,#c0392b);border-left:4px solid var(--red,#c0392b);border-radius:8px;padding:10px 14px;margin:0 0 16px;font-size:13px;line-height:1.5}
.rfr{display:flex;justify-content:flex-end;align-items:center;gap:8px;flex-wrap:wrap;margin:0 0 12px;font-size:12px;color:var(--muted,#5b6a67)}.rfr button{background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:6px 12px;font-size:12px;cursor:pointer}.rfr button:focus-visible{outline:2px solid var(--green,#2e7d32);outline-offset:2px}.rfr input{font:12px "Courier New",monospace;background:var(--card,#fff);color:var(--text,#1f2a28);border:1px solid var(--border,#dbe3e0);border-radius:6px;padding:5px 8px;width:300px;max-width:100%}
</style>
<div class="wrap">
<header><div><div class="eyebrow">COHESITY RECOVERABILITY EVIDENCE</div><h1>Recoverability Evidence Report</h1></div>
<div class="meta">Evidence ID <b id="eid"></b><br>Period <span id="per"></span><br>Generated {{GENERATED_UTC}}<br>Source: Helios MCP, read-only. Refreshes every 4 hours</div></header>
<!--HELIOS_ERR-->
<div class="rfr"><span id="rfn">Auto-refreshes every 4 hours.</span><button type="button" id="rf" title="Copy the command that pulls live data from Helios">Refresh from Helios</button></div>
<script>
(function(){
var CMD='Run the Cohesity-Recoverability-Evidence-Report skill to refresh this report from live Helios data';
var n=document.getElementById('rfn');
document.getElementById('rf').addEventListener('click',function(){
 function done(ok){if(ok){n.textContent='Copied. Paste it into Claude to pull live data; reopen or refresh this tab when it finishes.'}else{n.textContent='';var i=document.createElement('input');i.readOnly=true;i.value=CMD;i.setAttribute('aria-label','Refresh command');n.appendChild(document.createTextNode('Copy this into Claude; reopen or refresh this tab when it finishes: '));n.appendChild(i);i.focus();i.select()}}
 try{navigator.clipboard.writeText(CMD).then(function(){done(true)},function(){done(false)})}catch(e){done(false)}
});
})();
</script>

<div class="filter"><span class="lbl">Clusters</span>
<details class="ms" id="ms"><summary id="mssum">All clusters</summary>
<div class="menu"><div class="acts"><button type="button" id="selall">Select all</button><button type="button" id="selnone">Clear</button><button type="button" id="selconn">Connected only</button></div><div id="opts"></div></div></details>
<span class="sub" id="selnote"></span></div>

<div class="grid g4">
<div class="card"><div class="lbl">Groups in scope</div><div class="big" id="k1">-</div><div class="sub" id="k1s"></div><div class="tool">dataprotect-get-protection-groups</div></div>
<div class="card"><div class="lbl">Evidenced</div><div class="big ok" id="k2">-</div><div class="sub" id="k2s"></div><div class="tool">dataprotect-get-protection-group-runs</div></div>
<div class="card"><div class="lbl">Restorable recovery points</div><div class="big" id="k3">-</div><div class="sub" id="k3s"></div><div class="tool">dataprotect-get-protection-group-runs</div></div>
<div class="card"><div class="lbl">Evidence gaps</div><div class="big red" id="k4">-</div><div class="sub" id="k4s"></div><div class="tool">dataprotect-get-protection-policies</div></div>
</div>

<div class="card full"><h2>Summary Statement</h2><p class="summary" id="sm"></p></div>

<div class="grid g3">
<div class="card"><h2>Verdict By Cluster</h2><div class="legend"><span><i style="background:var(--green-lt)"></i>Evidenced</span><span><i style="background:var(--amber-lt)"></i>Partial</span><span><i style="background:var(--red-lt)"></i>Gap</span></div><div id="p1"></div><div class="foot" id="p1f"></div></div>
<div class="card"><h2>Verdict By Environment</h2><div class="legend"><span><i style="background:var(--green-lt)"></i>Evidenced</span><span><i style="background:var(--amber-lt)"></i>Partial</span><span><i style="background:var(--red-lt)"></i>Gap</span></div><div id="p2"></div></div>
<div class="card"><h2>Lowest Success Rate</h2><div class="tw"><table><thead><tr><th>Group</th><th>Cluster</th><th>Rate</th></tr></thead><tbody id="p3"></tbody></table></div></div>
</div>

<div class="card full"><div class="row"><h2>Evidence Register</h2><button type="button" class="btn" id="dl" hidden>Download evidence CSV</button></div>
<div class="fbar"><label for="vf">Verdict<select id="vf"><option value="all">All</option><option value="gap">Gap</option><option value="partial">Partial</option><option value="evidenced">Evidenced</option></select></label><label for="ve">Environment<select id="ve"><option value="">All</option></select></label><label for="vo">Offsite copy<select id="vo"><option value="">Any</option><option value="ok">Copy ok</option><option value="failed">Copy failed</option><option value="none">None</option></select></label><label for="vl">Immutable<select id="vl"><option value="">Any</option><option value="yes">Yes</option><option value="no">No</option></select></label><label for="vq">Search<input type="search" id="vq" placeholder="Group, cluster…"></label><label for="vs">Sort<select id="vs"><option value="worst">Worst first</option><option value="rate">Success % (low first)</option><option value="last">Last success (oldest first)</option><option value="points">Recovery points (fewest first)</option><option value="name">Group name</option></select></label><label for="vn">Rows<select id="vn"><option value="25">25</option><option value="50" selected>50</option><option value="100">100</option><option value="all">All</option></select></label></div>
<div class="tw sc"><table><thead><tr><th>Group</th><th>Cluster</th><th>Env</th><th>Runs</th><th>Success</th><th>Last success</th><th>Points</th><th>Oldest</th><th>Longest gap</th><th>Offsite</th><th>Immutable</th><th>Verdict</th></tr></thead><tbody id="rg"></tbody></table></div>
<div class="foot" id="rgf"></div></div>

<div class="card full"><h2>Gaps And Exceptions</h2>
<div class="tw sc"><table><thead><tr><th>Group</th><th>Cluster</th><th>Verdict</th><th>Reason</th></tr></thead><tbody id="gp"></tbody></table></div>
<div class="foot" id="gpf"></div></div>

<div class="card full"><h2>Limitations</h2>
<p>This report is derived from Helios backup run and snapshot metadata through a read-only connector. It shows that backups ran and succeeded, that restorable recovery points exist, how the schedule was met, and whether an offsite copy and immutability are configured. It does not show that a restore was performed or tested, and it is not a statement of regulatory compliance. Pair it with restore-test records when an auditor asks for them.</p></div>

<div class="foot">Filtering recalculates the cards, panels and tables in the page from the data collected at the generated time above; it does not query Helios live. Verdict rules: evidenced needs a restorable point, success at or above the limit, a recent success and no gap above the RPO tolerance; thresholds are a baseline, not a regulatory requirement. {{SCOPE_NOTE}}</div>
</div>

<script id="data" type="application/json">
{{DATA_JSON}}
</script>
<script>
(function(){
var D=JSON.parse(document.getElementById('data').textContent);
var sel={};D.clusters.forEach(function(c){sel[c.name]=true});
try{var s=JSON.parse(localStorage.getItem('crer-sel')||'null');if(s){D.clusters.forEach(function(c){if(c.name in s)sel[c.name]=!!s[c.name]})}}catch(e){}
var $=function(i){return document.getElementById(i)};
function fmt(n){return n.toLocaleString('en-US')}
function el(tag,cls,txt){var e=document.createElement(tag);if(cls)e.className=cls;if(txt!==undefined)e.textContent=txt;return e}
function age(h){if(h==null)return'none';if(h<48)return Math.round(h)+' h';return Math.round(h/24)+' d'}
function rate(r){return r==null?'-':r+'%'}
function sbar(name,ev,pa,gp){var r=el('div','bar');r.appendChild(el('div','n',name)).title=name;
 var t=el('div','t'),tot=ev+pa+gp;
 [[ev,'var(--green-lt)'],[pa,'var(--amber-lt)'],[gp,'var(--red-lt)']].forEach(function(x){var f=el('div','f');f.style.width=(tot?x[0]/tot*100:0)+'%';f.style.background=x[1];t.appendChild(f)});
 r.appendChild(t);r.appendChild(el('div','c',ev+' / '+tot));return r}
function tally(list,key){var m={};list.forEach(function(g){var k=g[key];m[k]=m[k]||{ev:0,pa:0,gp:0};
 if(g.status==='evidenced')m[k].ev++;else if(g.status==='partial')m[k].pa++;else m[k].gp++});return m}
function buildOpts(){var o=$('opts');o.textContent='';D.clusters.forEach(function(c){
 var l=el('label'),i=document.createElement('input');i.type='checkbox';i.checked=sel[c.name];
 i.addEventListener('change',function(){sel[c.name]=i.checked;save();render()});
 l.appendChild(i);l.appendChild(el('span','',c.name));l.appendChild(el('span','tag',c.connected?'connected':'not connected'));o.appendChild(l)})}
function save(){try{localStorage.setItem('crer-sel',JSON.stringify(sel))}catch(e){}}
function setAll(fn){D.clusters.forEach(function(c){sel[c.name]=fn(c)});save();buildOpts();render()}
var cur=[];
function csv(){var cols=['group','cluster','env','runs','ok','failed','rate','lastOkH','points','newestH','oldestD','gapH','offsite','lock','status','reason'];
 function q(v){v=v==null?'':String(v);return /[",\n]/.test(v)?'"'+v.replace(/"/g,'""')+'"':v}
 var out=['evidence_id,'+q(D.evidenceId)+',period_start,'+q(D.startUtc)+',period_end,'+q(D.endUtc),cols.join(',')];
 cur.forEach(function(g){out.push(cols.map(function(c){return q(g[c])}).join(','))});return out.join('\n')}
function render(){
 var S=D.clusters.filter(function(c){return sel[c.name]});
 var n=S.length,tot=D.clusters.length;
 $('eid').textContent=D.evidenceId;$('per').textContent=D.startUtc+' to '+D.endUtc+' ('+D.periodDays+' d)';
 $('mssum').textContent=n===tot?'All clusters ('+tot+')':n===0?'No clusters selected':n<=2?S.map(function(c){return c.name}).join(', '):n+' of '+tot+' clusters';
 $('selnote').textContent=n===0?'Select at least one cluster.':'';
 var G=D.groups.filter(function(g){return sel[g.cluster]});cur=G;
 var ev=G.filter(function(g){return g.status==='evidenced'}).length,pa=G.filter(function(g){return g.status==='partial'}).length,gp=G.length-ev-pa;
 var pts=0,runs=0,ok=0;G.forEach(function(g){pts+=g.points||0;runs+=g.runs||0;ok+=g.ok||0});
 $('k1').textContent=G.length;$('k1s').textContent='assessed in '+S.filter(function(c){return !c.na}).length+' cluster(s)';
 $('k2').textContent=G.length?Math.round(ev/G.length*100)+'%':'n/a';$('k2s').textContent=ev+' evidenced, '+pa+' partial';
 $('k3').textContent=fmt(pts);$('k3s').textContent=runs?Math.round(ok/runs*100)+'% of '+fmt(runs)+' runs succeeded':'no runs in period';
 $('k4').textContent=gp;$('k4s').textContent=gp?'no restorable point or stale':'none';
 var sm=G.length?'In the selected scope, '+ev+' of '+G.length+' protection groups ('+Math.round(ev/G.length*100)+'%) show recoverability evidence for '+D.startUtc+' to '+D.endUtc+': at least one restorable recovery point, a success rate of '+D.standards.minSuccessPct+'% or more, a recent success and no gap beyond '+D.standards.rpoTolerance+' times the policy interval. '+pa+' are partial and '+gp+' have a gap. '+fmt(pts)+' restorable recovery points were counted. This is derived from run metadata and does not include a restore test.':'No protection groups in the selection.';
 $('sm').textContent=sm;
 var p1=$('p1');p1.textContent='';var tc=tally(G,'cluster');
 S.filter(function(c){return !c.na}).forEach(function(c){var t=tc[c.name]||{ev:0,pa:0,gp:0};p1.appendChild(sbar(c.name,t.ev,t.pa,t.gp))});
 $('p1f').textContent=S.filter(function(c){return c.na}).map(function(c){return c.name+': n/a ('+c.na+')'}).join('. ');
 var p2=$('p2');p2.textContent='';var te=tally(G,'env');
 Object.keys(te).sort(function(a,b){return(te[b].ev+te[b].pa+te[b].gp)-(te[a].ev+te[a].pa+te[a].gp)}).forEach(function(k){p2.appendChild(sbar(k,te[k].ev,te[k].pa,te[k].gp))});
 if(!G.length)p2.appendChild(el('div','empty','No data for the selection.'));
 var p3=$('p3');p3.textContent='';
 G.filter(function(g){return g.rate!=null}).sort(function(a,b){return a.rate-b.rate}).slice(0,5).forEach(function(g){var r=el('tr');r.appendChild(el('td','',g.group));r.appendChild(el('td','',g.cluster));r.appendChild(el('td','num '+(g.rate<D.standards.minSuccessPct?'red':''),g.rate+'%'));p3.appendChild(r)});
 var ord={gap:0,partial:1,evidenced:2};
 var R=G.slice().sort(function(a,b){return ord[a.status]-ord[b.status]||(a.rate==null?-1:a.rate)-(b.rate==null?-1:b.rate)});
 var rg=$('rg');rg.textContent='';
 var vf=$('vf').value,fe=$('ve').value,fo=$('vo').value,fl=$('vl').value,fq=($('vq').value||'').toLowerCase(),fs=$('vs').value,fn=$('vn').value==='all'?1e9:parseInt($('vn').value,10);
 var RV=R.filter(function(g){return(vf==='all'||g.status===vf)&&(!fe||g.env===fe)&&(!fo||g.offsite===fo)&&(!fl||(fl==='yes')===!!g.lock)&&(!fq||(g.group+' '+g.cluster).toLowerCase().indexOf(fq)>=0)});
 function nz(v,d){return v==null?d:v}
 if(fs==='rate')RV.sort(function(a,b){return nz(a.rate,-1)-nz(b.rate,-1)});
 else if(fs==='last')RV.sort(function(a,b){return nz(b.lastOkH,1e9)-nz(a.lastOkH,1e9)});
 else if(fs==='points')RV.sort(function(a,b){return a.points-b.points});
 else if(fs==='name')RV.sort(function(a,b){return a.group<b.group?-1:1});
 RV.slice(0,fn).forEach(function(g){var r=el('tr');
  r.appendChild(el('td','',g.group+(g.paused?' (paused)':'')));r.appendChild(el('td','',g.cluster));r.appendChild(el('td','',g.env||''));
  r.appendChild(el('td','num',String(g.runs)));r.appendChild(el('td','num',rate(g.rate)));r.appendChild(el('td','num',age(g.lastOkH)));
  r.appendChild(el('td','num',String(g.points)));r.appendChild(el('td','num',g.oldestD==null?'none':g.oldestD+' d'));
  r.appendChild(el('td','num'+(g.rpoMet===false?' red':''),g.gapH==null?'-':age(g.gapH)));
  var o=el('td');o.appendChild(el('span','chip '+(g.offsite==='ok'?'ok':g.offsite==='failed'?'failed':''),g.offsite==='ok'?'copy ok':g.offsite==='failed'?'copy failed':'none'));r.appendChild(o);
  r.appendChild(el('td','',g.lock?'Yes':'No'));
  var v=el('td');v.appendChild(el('span','chip '+g.status,g.status));r.appendChild(v);rg.appendChild(r)});
 if(!RV.length){var e=el('tr'),d=el('td','empty','No protection groups match the selection and verdict.');d.colSpan=12;e.appendChild(d);rg.appendChild(e)}
 $('rgf').textContent='Showing '+Math.min(fn,RV.length)+' of '+RV.length+' matching groups ('+R.length+' in the cluster selection). The CSV contains all '+R.length+'.';
 var X=R.filter(function(g){return g.status!=='evidenced'});
 var gpb=$('gp');gpb.textContent='';
 X.slice(0,25).forEach(function(g){var r=el('tr');r.appendChild(el('td','',g.group));r.appendChild(el('td','',g.cluster));
  var v=el('td');v.appendChild(el('span','chip '+g.status,g.status));r.appendChild(v);r.appendChild(el('td','why',g.reason||''));gpb.appendChild(r)});
 if(!X.length){var e2=el('tr'),d2=el('td','empty','Every group in the selection is evidenced.');d2.colSpan=4;e2.appendChild(d2);gpb.appendChild(e2)}
 $('gpf').textContent=X.length>25?'Showing 25 of '+X.length+' groups without full evidence, worst first.':'';
}
(function(){var ev=[];D.groups.forEach(function(g){if(g.env&&ev.indexOf(g.env)<0)ev.push(g.env)});ev.sort().forEach(function(v){var o=el('option','',v);o.value=v;$('ve').appendChild(o)});
 ['vf','ve','vo','vl','vs','vn'].forEach(function(i){$(i).addEventListener('change',render)});$('vq').addEventListener('input',render)})();
$('selall').addEventListener('click',function(){setAll(function(){return true})});
$('selnone').addEventListener('click',function(){setAll(function(){return false})});
$('selconn').addEventListener('click',function(){setAll(function(c){return c.connected})});
document.addEventListener('click',function(e){var d=$('ms');if(d.open&&!d.contains(e.target))d.open=false});
buildOpts();render();
(async function(){try{var dl=await claude.use('downloads');if(!dl)return;var b=$('dl');b.hidden=false;
 b.addEventListener('click',async function(){try{await dl.save({filename:D.evidenceId+'.csv',data:csv()})}catch(e){}})}catch(e){}})();
})();
</script>
```
