# SpectreAI — Maestro Orchestration Layer

This repository contains the **UiPath Maestro orchestration layer** for SpectreAI — an AI-powered multi-agent system that turns a Slack support message into a diagnosed, patched, and PR-raised fix in under 2 minutes.

**UiPath AgentHack 2026 submission** — Nithin BR, Agentic Architect @ Persistent Systems

---

## Project Description

SpectreAI eliminates the manual RPA support loop. When a bot fails in production, a developer typically spends 2–4 hours digging through Orchestrator logs, finding the broken XAML line, patching it, and raising a PR — all while the end user waits with zero visibility.

SpectreAI automates that entire loop:

1. End user fills a Slack shortcut form — process number, issue type, transaction ID, description, impact
2. Maestro picks up the structured message, classifies it as a bug or enhancement, and routes it to the right agent
3. For bugs: InvestigationAgent runs a 3-layer log fetch, queries SpectreKB, and produces a confidence-scored diagnosis
4. For high-confidence bugs: CodingAgent patches the XAML via GitHub API (no cloning), validates the XML, and raises a Draft PR
5. For low-confidence cases: Slack interactive buttons let the user escalate to human review via Action Centre
6. For enhancements: Jira ticket auto-created from the Slack form, confirmation sent back to user

**Three paths fully automated:** Happy Path (code fix), Escalation Path (human review), Enhancement Path (Jira ticket).

---

## UiPath Components Used

| Component | Usage |
|---|---|
| **UiPath Maestro** | BPMN orchestration — classifies bug vs enhancement, routes to agents, handles all conditional logic |
| **UiPath Coded Agents (Python)** | SpectreInvestigationAgent and SpectreCodingAgent — both implemented as Python Coded Agents |
| **UiPath Orchestrator** | Log fetch API (job logs, business exception logs, transaction item logs), asset management |
| **UiPath Integration Service** | Slack connector (trigger + notifications), Jira connector (ticket creation) |
| **UiPath Action Centre** | Human review task raised on escalation path with full diagnosis pre-filled |
| **UiPath Apps** | Spectre Escalation Review app — Accept/Dismiss UI rendered in Action Centre |

---

## Agent Type

SpectreAI uses **both Coded Agents and Maestro orchestration**:

- **SpectreInvestigationAgent** — Python Coded Agent. Fetches 3 layers of Orchestrator logs, queries SpectreKB knowledge base, calls GPT-4.1 mini reasoning engine, returns structured diagnosis with confidence score.
- **SpectreCodingAgent** — Python Coded Agent. Fetches XAML via GitHub Contents API (no cloning), produces surgical patch via reasoning engine, validates XML, commits and raises Draft PR via GitHub API.
- **Maestro (BPMN)** — Low-code orchestration layer. Handles all routing logic, triggers agents, manages Slack notifications, creates Jira tickets, raises Action Centre tasks.

---

## Repository Structure

```
Specter1/
├── Agentic Process/          # Maestro BPMN orchestration process
│   ├── SpectreAI.bpmn        # Core BPMN workflow — all routing logic
│   ├── bindings_v2.json      # Resource bindings (Slack trigger, Jira, agents, app)
│   ├── entry-points.json     # Process entry point definition
│   └── project.uiproj        # UiPath project file
│
├── Spectre Escalation Review/ # UiPath App — Action Centre human review form
│   ├── Main.xaml             # App entry point
│   ├── BotFailureDiagnosisPagePage_Accept_click.xaml  # Accept action handler
│   └── project.json          # App project definition and dependencies
│
├── resources/
│   └── solution_folder/      # Solution deployment manifests
│       ├── connection/       # Slack and Jira connection configurations
│       ├── package/          # Agent package references
│       └── process/          # Process and app deployment configs
│
└── SolutionStorage.json      # Solution metadata
```

---

## Architecture Overview

```
Slack Message
     │
     ▼
[Maestro BPMN — SpectreAI.bpmn]
     │
     ├── Enhancement? ──────────────────► Create Jira Ticket → Slack confirmation → End
     │
     └── Bug?
          │
          ▼
     [SpectreInvestigationAgent — Python Coded Agent]
     3-layer log fetch + SpectreKB lookup
     Structured diagnosis + confidence score
          │
          ├── No Error / Business Exception ──► Dismiss → Notify user → End
          │
          ├── Low Confidence ────────────────► Escalate
          │                                        │
          │                                   [Spectre Escalation Review App]
          │                                   Jira bug ticket + Action Centre task
          │                                   Human reviews + accepts/dismisses
          │
          └── High Confidence Bug ───────────► [SpectreCodingAgent — Python Coded Agent]
                                                   │
                                                   ├── Fix Done ────► Commit + Draft PR
                                                   │                       │
                                                   │                  Support Channel + Dev Channel notified
                                                   │
                                                   └── Fix Not Done ► Report-only PR
```

---

## Setup Instructions

### Prerequisites

- UiPath Studio 2024.10+ with Maestro enabled
- UiPath Orchestrator (cloud) with Maestro licence
- UiPath Integration Service — Slack connector configured
- UiPath Integration Service — Jira connector configured
- `SpectreInvestigationAgent` deployed as a Function process in Orchestrator
- `SpectreCodingAgent` deployed as a Function process in Orchestrator
- GitHub PAT with repo read/write access
- Slack Bot Token with channels:read, chat:write, shortcuts permissions

### Step 1 — Clone and open the solution

```bash
git clone https://github.com/UipathAgentHackNithin/SpectreAI-Maestro.git
```

Open `Specter1` as a Solution in UiPath Studio (File → Open Solution). Studio will load both the Agentic Process and the Escalation Review app.

### Step 2 — Configure Orchestrator assets

Create the following assets in UiPath Orchestrator before running:

| Asset Name | Used By | Value |
|---|---|---|
| `GITHUB_PAT` | SpectreCodingAgent | GitHub Personal Access Token |
| `SLACK_BOT_TOKEN` | Both agents | Slack Bot OAuth token |
| `SLACK_SUPPORT_CHANNEL_ID` | Both agents | Slack support channel ID |
| `SLACK_DEV_CHANNEL_ID` | SpectreCodingAgent | Slack dev channel ID |
| `JIRA_PROJECT_KEY` | Maestro + InvestigationAgent | Jira project key (e.g. `SAI`) |
| `ORCHESTRATOR_BASE_URL` | InvestigationAgent | Your Orchestrator cloud URL |
| `UIPATH_ACCESS_TOKEN` | InvestigationAgent | Orchestrator API access token |

### Step 3 — Configure Integration Service connections

1. In UiPath Integration Service, create a **Slack** connection
2. In UiPath Integration Service, create a **Jira** connection
3. Update `bindings_v2.json` with your connection IDs if deploying to a different tenant

### Step 4 — Deploy the Python Coded Agents

Clone and deploy both agents separately:

- [SpectreInvestigationAgent](https://github.com/UipathAgentHackNithin/SpectreInvestigationAgent) — deploy as a Function process
- [SpectreCodingAgent](https://github.com/UipathAgentHackNithin/SpectreCodingAgent) — deploy as a Function process

Follow the README in each repo for agent-specific setup (Python dependencies, SpectreKB configuration).

### Step 5 — Publish the Maestro solution

Publish the full solution from UiPath Studio to Orchestrator. This deploys:
- Agentic Process (Maestro BPMN)
- Spectre Escalation Review app
- All resource bindings

### Step 6 — Configure the Slack trigger

1. In Orchestrator, activate the Slack event trigger on the Agentic Process
2. The trigger fires on messages containing `SPECTRE_TICKET`
3. Set up the **"SpectreAI Bot Support Request"** Slack shortcut in your workspace
4. The shortcut form collects: Process Number, Issue Type, Transaction ID, Description, Impact

### Step 7 — Test the solution

Submit a test form via the Slack shortcut and verify:
- Maestro job triggers in Orchestrator
- For a Bug submission: InvestigationAgent job starts, diagnosis appears in Slack thread
- For high confidence: CodingAgent job starts, Draft PR appears on GitHub
- For an Enhancement submission: Jira ticket created, Slack confirmation sent

---

## Related Repositories

| Repository | Description |
|---|---|
| [SpectreAI-Maestro](https://github.com/UipathAgentHackNithin/SpectreAI-Maestro) | This repo — Maestro orchestration, BPMN, Action Centre app |
| [SpectreInvestigationAgent](https://github.com/UipathAgentHackNithin/SpectreInvestigationAgent) | Python Coded Agent — 3-layer log fetch, SpectreKB, reasoning engine diagnosis |
| [SpectreCodingAgent](https://github.com/UipathAgentHackNithin/SpectreCodingAgent) | Python Coded Agent — XAML patch, GitHub API, XML validation, Draft PR |
| [InvoiceProcessing-Performer](https://github.com/UipathAgentHackNithin/InvoiceProcessing-Performer) | Sample target bot used in demo |

---

## Demo

- **Demo Video:** https://www.youtube.com/watch?v=d64LqEl6M5Y
- **Devpost:** https://devpost.com/software/zeroday
- **UiPath Forum:** https://forum.uipath.com/t/spectreai-from-bots-down-in-slack-to-a-draft-pr-in-under-2-minutes-agenthack-2026/5755787

---

## Author

Nithin BR — Agentic Architect @ Persistent Systems
UiPath AgentHack 2026
