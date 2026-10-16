# Sample Output: Cohesity Recoverability Evidence Agent

Prompt: `Produce a recoverability evidence report for the last 7 days for cluster statngce741`

Model: GPT-5 Chat. Lab data. Captured in the Copilot Studio test pane in Light mode.

![Recoverability Evidence agent answer in Copilot Studio](../docs/screenshots/recoverability-evidence-report-copilot.png)

## Notes

| Topic | Note |
|---|---|
| Format | The answer is chat text with tables. The Claude skill gives a live, filterable page. See the screenshot in the main README. |
| Scope | Name one cluster and a short period. A 30 day report on a large cluster can exceed the tool limits. |
| Evidence | The report is built from run metadata. It does not prove that a restore works. |
| Accuracy | The agent follows the instructions but is a language model. Compare a few answers with Helios before you share the agent. |
| Your data | Cluster and group names in the image are lab data. Your answers will differ. |
