# Copilot Studio Lab Guide: Skills, Triggers, Hooks, Workflow Automation, Agent-to-Agent, and Dataverse

> Hands-on lab in a module format inspired by Agent Academy learning guides.

## Table of contents

1. [Overview](#overview)
2. [Learning outcomes](#learning-outcomes)
3. [Prerequisites](#prerequisites)
4. [Architecture](#architecture)
5. [Module 1 - Create the base agent](#module-1---create-the-base-agent)
6. [Module 2 - Add skills, tools, and trigger](#module-2---add-skills-tools-and-trigger)
7. [Module 3 - Add Session Workflow Hook (global guardrail)](#module-3---add-session-workflow-hook-global-guardrail)
8. [Module 4 - Add Python Hook (per-message dynamic guardrail)](#module-4---add-python-hook-per-message-dynamic-guardrail)
9. [Module 5 - Add workflow automation for HR case intake](#module-5---add-workflow-automation-for-hr-case-intake)
10. [Module 6 - Add agent-to-agent scenario](#module-6---add-agent-to-agent-scenario)
11. [Module 7 - Add Dataverse storage and auto-update flow](#module-7---add-dataverse-storage-and-auto-update-flow)
12. [Module 8 - Visualize and test hook differences](#module-8---visualize-and-test-hook-differences)
13. [Troubleshooting](#troubleshooting)
14. [Knowledge check](#knowledge-check)

---

## Overview

In this lab, you build an **HR Assist Agent** and progressively add advanced capabilities:

- Structured behavior with skills
- Trigger configuration
- Two hooks with different purposes:
  - **Session workflow hook** (runs once per chat session)
  - **Python hook** (runs every user message)
- Workflow-based task automation
- Agent-to-agent delegation
- Dataverse persistence and change-driven automation

---

## Learning outcomes

After this lab, you can:

1. Explain when to use a session hook vs Python hook
2. Build and test both hook types
3. Implement a practical automation workflow
4. Connect specialist agents for delegated responses
5. Persist case data and react to changes automatically

---

## Prerequisites

You need:

1. Copilot Studio access with maker rights
2. Ability to create/edit:
   - Agents
   - Workflows (Power Automate)
   - Dataverse tables and flows
3. Optional:
   - Teams/Outlook connectors
   - SharePoint site for artifacts

---

## Architecture

```text
User
  -> HR Assist Agent
      -> Session Hook (workflow, once per session)
      -> Python Hook (per user message)
      -> Workflow (case intake + escalation)
      -> Connected Payroll Agent (delegation path)
      -> Dataverse (create/update case records)
          -> Dataverse change-trigger flow (auto actions)
```

---

## Module 1 - Create the base agent

### Goal

Create the core `HR Assist Agent` with clear instructions.

### Steps

1. Create a new agent in Copilot Studio named `HR Assist Agent`.
2. Open **Build**.
3. Add this baseline instruction:

```text
You are HR Assist Agent, a professional HR support assistant.
Handle onboarding, records requests, manager workflows, policy questions, and payroll FAQs.

Rules:
- Be concise, empathetic, and professional.
- Confirm employee identity before taking action.
- Do not invent policy numbers, thresholds, or legal conclusions.
- Offer escalation when risk/sensitivity is high.
```

### Checkpoint

- Agent exists and instruction text is saved.

### Screenshot

![Agent Build](images/01-agent-build.png)

---

## Module 2 - Add skills, tools, and trigger

### Goal

Add reusable capability blocks and ensure the chat trigger is available.

### Steps

1. Add skills:
   - `employee-onboarding`
   - `employee-records-request`
   - `manager-hr-workflows`
   - `hr-policy-qa`
2. Add tools:
   - Send email
   - Read SharePoint list items
   - Create a file in SharePoint
3. In **Triggers**, verify `Chat with the agent`.

### Checkpoint

- Skills and tools appear in Build panel.
- Chat trigger is active.

---

## Module 3 - Add Session Workflow Hook (global guardrail)

### Goal

Create a **global session guardrail** that runs once at session start.

### Why this hook exists

Use this when you need conversation-wide defaults every time:

- identity confirmation policy
- escalation style
- anti-hallucination constraints

### Steps

1. Build -> **More options (...)** -> **Hooks (Preview)**.
2. Create hook:
   - **Name:** `Session Start - Safety and Triage Context`
   - **When:** `Start (pre-loop)`
   - **Action:** `Run a workflow`
   - **Failure mode:** `Continue`
3. Create/select workflow: `Hook - SessionStart Safety and Triage`.
4. In that workflow:
   - Trigger: `When an agent calls the workflow`
   - Respond output: `additionalContext`
   - Return a policy context string for the session

### Example output text for `additionalContext`

```text
Always verify employee identity before actions.
Do not provide legal conclusions.
Do not invent policy numbers if unknown.
Use sensitive-case handling for discrimination/harassment complaints.
```

### Checkpoint

- Hook appears in list as Start (pre-loop) + Run a workflow.

---

## Module 4 - Add Python Hook (per-message dynamic guardrail)

### Goal

Create a **dynamic per-message guardrail** for risky prompts.

### Why this hook exists

Use Python hook when logic must inspect each message and branch conditionally.

### Steps

1. In Hooks, create:
   - **Name:** `Prompt Risk Detector (Python)`
   - **When:** `User prompt submitted`
   - **Action:** `Run a Python script`
   - **Failure mode:** `Continue`
2. Paste:

```python
def main(event):
    prompt = (event.get("prompt") or "").strip()
    if not prompt:
        return {"modifiedPrompt": prompt}

    risk_terms = [
        "discrimination",
        "harassment",
        "retaliation",
        "payroll dispute",
        "wrong salary",
        "legal",
        "lawsuit",
        "whistleblower"
    ]

    lowered = prompt.lower()
    has_risk = any(term in lowered for term in risk_terms)

    if not has_risk:
        return {"modifiedPrompt": prompt}

    safety_prefix = (
        "[HOOK:HIGH_RISK_HR_CASE]\n"
        "Handle with care: acknowledge empathy, avoid legal conclusions, "
        "ask for identity confirmation, summarize facts, and offer escalation to HR Operations.\n\n"
    )

    return {"modifiedPrompt": safety_prefix + prompt}
```

### Checkpoint

- Hook list shows:
  - Session Start - Safety and Triage Context (workflow)
  - Prompt Risk Detector (Python)

### Screenshots

![Hooks List](images/02-hooks-list.png)
![Add Hook Python](images/03-add-hook-python.png)

---

## Module 5 - Add workflow automation for HR case intake

### Goal

Automate case creation + escalation notification.

### Steps

Create flow `HR Case Intake and Escalation`:

1. Trigger: `When an agent calls the workflow`
2. Inputs:
   - `employeeId`
   - `issueType`
   - `summary`
   - `severity`
3. Actions:
   - Create Dataverse row (or SharePoint row for starter)
   - Send notification to HR Ops
4. Output back to agent:
   - `caseId`
   - `createdAt`
   - `escalationStatus`

### Workflow logic

Input capture -> record creation -> notification -> response to agent.

### Checkpoint

- Asking the agent to open a case returns a case ID.

---

## Module 6 - Add agent-to-agent scenario

### Goal

Delegate payroll-heavy requests to a specialist agent.

### Steps

1. Create second agent: `Payroll Specialist Agent`.
2. In HR agent -> **Connected agents** -> add Payroll agent.
3. Add routing rule in instruction/skill:

```text
If request is payroll-specific (tax, salary mismatch, arrears, bonus formula),
route to Payroll Specialist Agent.
```

### Checkpoint

- Payroll-specific query follows delegated specialist path.

---

## Module 7 - Add Dataverse storage and auto-update flow

### Goal

Persist case data and react automatically to updates.

### Steps

1. Create Dataverse table `hr_case` with:
   - `case_id`
   - `employee_id`
   - `category`
   - `severity`
   - `status` (New/In Review/Escalated/Closed)
   - `owner`
   - `last_updated`
2. Create automation flow:
   - Trigger: `When a row is added, modified or deleted`
   - If `status = Escalated` -> notify HR leadership
   - If `status = Closed` -> notify requester + update summary table

### Checkpoint

- Manual status update triggers downstream automation.

---

## Module 8 - Visualize and test hook differences

This section answers: **Why both hooks?**

## Core difference

| Hook | Runs when | Scope | Best use |
|---|---|---|---|
| Session workflow hook | Once at new chat start | Whole session | Set baseline policy/guardrail |
| Python hook | Every user message | Per-turn | Dynamic text detection and conditional rewrite |

## Practical examples

### Example A - Session hook value

User asks: `What is leave carry-forward?`  
Even without risky terms, session policy still applies (identity/precision/escalation style baseline).

### Example B - Python hook value

User asks: `My manager is discriminating and I may take legal action.`  
Per-message detection triggers prompt rewrite and safer handling for this turn.

### Example C - Why both together

Same chat:
1. Safe prompt
2. Risky prompt
3. Safe prompt again

Result:
- Session baseline stays active for all turns
- Python risk mode activates only on risky turn(s)

## Visual debug method (recommended)

Add temporary debug markers:

1. Session workflow output prefix: `[DEBUG:SESSION_HOOK]`
2. Python rewrite prefix: `[DEBUG:PY_HOOK]`

Then test:

1. New chat, safe prompt -> session marker behavior only
2. Risky prompt -> Python marker appears
3. Safe prompt again -> Python marker gone; session baseline remains

Remove markers after demo.

## Execution-count proof

In activity/monitor views:

- Session workflow hook should run **1 time per new chat**
- Python hook should run **1 time per user message**

---

## Troubleshooting

### Hook did not trigger

- Verify hook event (`When`) is correct
- Verify action type is correct
- Save + publish agent

### Python hook not changing behavior

- Ensure `def main(event)` exists
- Ensure return uses `{"modifiedPrompt": ...}`
- Keep failure mode as `Continue` during testing

### Workflow output missing

- Configure `Respond to the agent` outputs
- Match output names exactly
- Republish workflow

### Dataverse flow not firing

- Check trigger table and operation scope
- Validate connection permissions
- Test with direct row update

---

## Knowledge check

1. When would you use only session hook and skip Python hook?  
2. Why is Python hook better for keyword/risk scoring?  
3. What confirms both hooks are working (besides chat output)?  
4. What are the minimum fields needed to make a useful `hr_case` table?

---

## Next steps

1. Add weighted risk scoring in Python hook
2. Add human approval for High severity
3. Add monitoring dashboard for hook/workflow failures
4. Move constants to environment variables
