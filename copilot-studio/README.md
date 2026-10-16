# Cohesity Recoverability Evidence Agent For Microsoft Copilot Studio

Recoverability evidence over a reporting period, as a Copilot Studio agent. This is the same logic as the Claude skill in `../claude/`, rewritten as agent instructions and knowledge files.

## What is in this folder

| File | Where it goes in Copilot Studio |
|---|---|
| `instructions.md` | Agent, Overview, Instructions (under the 8,000 character limit) |
| `knowledge/evidence-report-methodology.md` | Agent, Knowledge, Add knowledge, Upload file |
| `knowledge/helios-mcp-tool-notes.md` | Agent, Knowledge, Add knowledge, Upload file |
| `starters.md` | Agent, Overview, Suggested prompts (one per line) |

## Setup

1. In Copilot Studio, choose **Agents**, then **Create blank agent** (or **New agent**), and create an agent named **Cohesity Recoverability Evidence**.
2. On **Overview**, under **Details**, choose **Edit** and paste the one-sentence description from the top of this README into **Description** (available after the agent has finished setting up).
3. Under **Instructions**, choose **Edit** and paste the full content of `instructions.md`.
4. Under **Knowledge**, choose **Add knowledge**, **Upload file**, and upload both files from `knowledge/`.
5. Under **Tools**, choose **Add tool**, **New tool**, **Model Context Protocol**, and connect your Helios MCP server (the same server the Claude skill uses as `helios-mcp`). Enable only the read-only tools listed in `knowledge/helios-mcp-tool-notes.md`. Leave write or action tools off.
6. Keep **Web Search** disabled in the **Knowledge** section (the default), so answers come from Helios and the uploaded files.
7. Under **Suggested prompts**, choose **Add suggested prompts** and add the prompts from `starters.md`:

- Produce a recoverability evidence report for the last 30 days
- Which protection groups have an evidence gap and why?
- Show recovery point evidence for <cluster name>
- Evidence report for the last 90 days with a 98% success threshold
- Which groups have no offsite copy or no immutability?

8. Choose **Test** and try a prompt in the test pane. Then choose **Publish** and add the channels you need on the **Channels** page (Teams, Microsoft 365 Copilot).

## Differences from the Claude skill

- **No published page.** The Claude skill publishes a live, filterable page that refreshes every 4 hours. The Copilot Studio agent answers in chat with tables and runs on demand. For the page, use the Claude skill.
- **No cluster multi-select.** Name the cluster in your question instead.
- **Scheduling.** To get a regular result, create a Power Automate flow with a schedule trigger that calls the agent and posts the answer to Teams or email. This is not part of this folder.
- **Large Helios responses.** The Claude skill processes large responses with code. In Copilot Studio, long runs can be slow or hit tool limits on big estates. Ask for one cluster at a time, or enable the **Code interpreter** if your tenant offers it.
- **No CSV download button.** Ask for a CSV block in chat and copy it.
- **Read-only.** The agent never changes anything in Helios.

## Check before you publish

Share the agent only with people who may see backup posture data for the connected clusters. Ask a few known questions first and compare the answers with Helios, because the agent follows the instructions but is a language model, not a fixed report.
