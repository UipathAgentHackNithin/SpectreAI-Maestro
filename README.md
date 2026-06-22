# SpectreAI — Maestro Orchestration Layer

This repository contains the **UiPath Maestro orchestration layer** for SpectreAI — the AI-powered RPA bot self-healing system built for UiPath AgentHack 2025.

SpectreAI turns a Slack support message into a diagnosed, patched, and PR-raised fix in under 2 minutes — fully automated, zero developer intervention for high-confidence bugs.

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

SpectreAI is a **multi-agent agentic system** orchestrated by UiPath Maestro. This repository contains the orchestration brain — the BPMN process that decides what happens, when, and in what order.

```
Slack Message
     │
     ▼
[Maestro BPMN — SpectreAI.bpmn]
     │
     ├── Enhancement? ──────────────────► Create Jira Ticket → End
     │
     └── Bug?
          │
          ▼
     [SpectreInvestigationAgent]
     3-layer log fetch + SpectreKB lookup
     Structured diagnosis + confidence score
          │
          ├── No Error / Business Exc. ──► Dismiss → Notify user → End
          │
          ├── Low Confidence ────────────► Escalate
          │                                    │
          │                               [Spectre Escalation Review App]
          │                               Action Centre task created
          │                               Human reviews + accepts/dismisses
          │
          └── High Confidence Bug ───────► [SpectreCodingAgent]
                                               │
                                               ├── Fix Done ────► Commit + Draft PR
                                               │                       │
                                               │                  Support Channel
                                               │                  Dev Channel
                                               │
                                               └── Fix Not Done ► Report-only PR
```

---

## Components

### 1. Agentic Process (BPMN Orchestration)

The core of SpectreAI. A **UiPath Maestro ProcessOrchestration** project containing the full BPMN workflow (`SpectreAI.bpmn`).

**Trigger:** Slack event listener — fires when a message containing `SPECTRE_TICKET` appears in the support channel. This is the output of the SpectreAI Bot Support Request Slack shortcut.

**Routing logic:**
- Classifies issue as Bug or Enhancement based on form input
- Routes Enhancement directly to Jira ticket creation via Integration Service
- Routes Bug to `SpectreInvestigationAgent` for diagnosis
- Routes based on confidence score:
  - High confidence code bug → `SpectreCodingAgent`
  - Low confidence / ambiguous → `Spectre Escalation Review` app (Action Centre)
  - Business exception / no error → Dismiss with user notification

**Resource bindings (`bindings_v2.json`):**
| Resource | Type | Purpose |
|---|---|---|
| uipath-salesforce-slack | Connection | Slack event trigger + notifications |
| uipath-atlassian-jira | Connection | Enhancement ticket creation |
| SpectreInvestigationAgent | Function Process | 3-layer log investigation |
| SpectreCodingAgent | Function Process | XAML patching + PR creation |
| Spectre Escalation Review | UiPath App | Action Centre human review form |

---

### 2. Spectre Escalation Review (UiPath App)

A **UiPath App** that renders in Action Centre when the investigation confidence is too low to auto-fix.

**What it shows:**
- Process name and number
- Transaction ID
- Full diagnosis from InvestigationAgent (root cause, confidence score, recommended action)
- Accept / Dismiss action buttons

**What it does:**
- **Accept** — acknowledges the escalation, notifies the support channel that a human is reviewing
- **Dismiss** — closes the task, notifies the user that no action is needed

**Dependencies:**
- `UiPath.IntegrationService.Activities` 1.27.0
- `UiPath.DataService.Activities` 25.9.3
- `UiPath.WorkflowEvents.Activities` 3.39.0

---

### 3. Solution Resources

The `resources/solution_folder/` directory contains deployment manifests for the full SpectreAI solution:

| Manifest | Purpose |
|---|---|
| `connection/uipath-salesforce-slack/` | Slack Integration Service connection config |
| `connection/uipath-atlassian-jira/` | Jira Integration Service connection config |
| `package/SpectreInvestigationAgent.json` | Investigation agent package reference |
| `package/SpectreCodingAgent.json` | Coding agent package reference |
| `package/Agentic_Process.json` | Orchestration process package reference |
| `process/function/SpectreInvestigationAgent.json` | Investigation agent deployment config |
| `process/function/SpectreCodingAgent.json` | Coding agent deployment config |
| `process/processOrchestration/Agentic_Process.json` | Orchestration process deployment config |

---

## Related Repositories

SpectreAI is a multi-repo solution. This repository contains the orchestration layer only.

| Repository | Description |
|---|---|
| [SpectreAI-Maestro](https://github.com/UipathAgentHackNithin/SpectreAI-Maestro) | This repo — Maestro orchestration, BPMN, Action Centre app |
| [SpectreInvestigationAgent](https://github.com/UipathAgentHackNithin/SpectreInvestigationAgent) | Python coded agent — 3-layer log fetch, SpectreKB, LLM diagnosis |
| [SpectreCodingAgent](https://github.com/UipathAgentHackNithin/SpectreCodingAgent) | Python coded agent — XAML patch, GitHub API, Draft PR |
| [InvoiceProcessing-Performer](https://github.com/UipathAgentHackNithin/InvoiceProcessing-Performer) | Sample target bot used in demo |

---

## Setup & Deployment

### Prerequisites

- UiPath Studio 2024.10+
- UiPath Orchestrator (cloud) with Maestro enabled
- UiPath Integration Service — Slack connector configured
- UiPath Integration Service — Jira connector configured
- `SpectreInvestigationAgent` deployed as a Function process in Orchestrator
- `SpectreCodingAgent` deployed as a Function process in Orchestrator

### Deployment Steps

1. **Clone this repository**
   ```bash
   git clone https://github.com/UipathAgentHackNithin/SpectreAI-Maestro.git
   ```

2. **Open in UiPath Studio**
   - Open `Specter1` as a Solution (File → Open Solution)
   - Studio will load both the Agentic Process and the Escalation Review app

3. **Configure connections**
   - In UiPath Integration Service, create a Slack connection and note the connection ID
   - In UiPath Integration Service, create a Jira connection and note the connection ID
   - Update `bindings_v2.json` with your connection IDs if deploying to a different tenant

4. **Publish the solution**
   - Publish the full solution from Studio to Orchestrator
   - This deploys: Agentic Process, Spectre Escalation Review app, and all resource bindings

5. **Configure the Slack trigger**
   - In Orchestrator, activate the Slack event trigger on the Agentic Process
   - The trigger fires on messages containing `SPECTRE_TICKET` (output of the Slack shortcut form)

6. **Deploy the Slack shortcut**
   - Set up the "SpectreAI Bot Support Request" Slack shortcut in your workspace
   - The shortcut form collects: Process Number, Issue Type, Transaction ID, Description, Impact
   - On submit, it posts a structured message to the support channel with `SPECTRE_TICKET` tag

7. **Test**
   - Submit a test form via the Slack shortcut
   - Verify the Maestro process triggers and routes correctly
   - Check Orchestrator job logs for the full execution trace

---

## Environment Variables / Assets

The following Orchestrator assets must be configured before running:

| Asset | Used By | Purpose |
|---|---|---|
| `GITHUB_PAT` | SpectreCodingAgent | GitHub API access for repo discovery and PR creation |
| `SLACK_BOT_TOKEN` | Both agents | Slack API for status updates |
| `SLACK_SUPPORT_CHANNEL_ID` | Both agents | Support channel for user notifications |
| `SLACK_DEV_CHANNEL_ID` | SpectreCodingAgent | Dev channel for PR notifications |
| `JIRA_PROJECT_KEY` | Maestro + InvestigationAgent | Jira project for ticket creation |
| `ORCHESTRATOR_BASE_URL` | InvestigationAgent | Orchestrator API for log fetching |

---

## How It Fits Together

```
User fills Slack shortcut form
        │
        ▼
Slack posts SPECTRE_TICKET message
        │
        ▼
Maestro trigger fires (this repo)
        │
        ▼
BPMN routes to InvestigationAgent ──► SpectreInvestigationAgent repo
        │
        ▼
BPMN routes to CodingAgent ──────────► SpectreCodingAgent repo
        │
        ▼
BPMN routes to Action Centre ────────► Spectre Escalation Review app (this repo)
        │
        ▼
Slack notifications sent
```

---

## Author

Nithin BR — Agentic Architect @ Persistent Systems
UiPath AgentHack 2025 submission
