# Cohesity Recoverability Evidence Agent For Microsoft Copilot Studio

Recoverability evidence over a reporting period, as a Copilot Studio agent. This is the same logic as the Claude skill in `../claude/`, rewritten as agent instructions and knowledge files.

## What is in this folder

| File | Where it goes in Copilot Studio |
|---|---|
| `instructions.md` | Agent, Overview, Instructions (under the 8,000 character limit) |
| `knowledge/evidence-report-methodology.md` | Agent, Knowledge, Add knowledge, Files |
| `knowledge/helios-mcp-tool-notes.md` | Agent, Knowledge, Add knowledge, Files |
| `starters.md` | Agent, Overview, Conversation starters (one per line) |

## Setup

1. In Copilot Studio, create a new agent named **Cohesity Recoverability Evidence**. Paste the description above into the agent description.
2. Open **Overview**, choose **Edit** under Instructions, and paste the full content of `instructions.md`.
3. Open **Knowledge**, choose **Add knowledge**, **Files**, and upload both files from `knowledge/`.
4. Open **Tools**, choose **Add a tool**, **Model Context Protocol**, and connect your Helios MCP server (the same server the Claude skill uses as `helios-mcp`). Enable only the read-only tools listed in `knowledge/helios-mcp-tool-notes.md`. Leave write or action tools off.
5. In **Settings**, turn **Use general knowledge** off and keep the web search off, so answers come only from Helios and the uploaded files.
6. Add the conversation starters from `starters.md`:

- Produce a recoverability evidence report for the last 30 days
- Which protection groups have an evidence gap and why?
- Show recovery point evidence for <cluster name>
- Evidence report for the last 90 days with a 98% success threshold
- Which groups have no offsite copy or no immutability?

7. Test in the test pane, then publish and add the channels you need (Teams, Microsoft 365 Copilot).

## Differences from the Claude skill

- **No published page.** The Claude skill publishes a live, filterable page that refreshes every 4 hours. The Copilot Studio agent answers in chat with tables and runs on demand. For the page, use the Claude skill.
- **No cluster multi-select.** Name the cluster in your question instead.
- **Scheduling.** To get a regular result, create a Power Automate flow with a schedule trigger that calls the agent and posts the answer to Teams or email. This is not part of this folder.
- **Large Helios responses.** The Claude skill processes large responses with code. In Copilot Studio, long runs can be slow or hit tool limits on big estates. Ask for one cluster at a time, or enable the **Code interpreter** if your tenant offers it.
- **No CSV download button.** Ask for a CSV block in chat and copy it.
- **Read-only.** The agent never changes anything in Helios.

## Check before you publish

Share the agent only with people who may see backup posture data for the connected clusters. Ask a few known questions first and compare the answers with Helios, because the agent follows the instructions but is a language model, not a fixed report.
