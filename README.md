# Hands-On Lab Guide: Copilot Studio Agent with Skills, Triggers, Hooks, Workflow Automation, Agent-to-Agent, and Dataverse Auto-Update

## Lab at a glance

- **Level:** Beginner to Intermediate  
- **Mode:** Fully hands-on (build while you read)  
- **Estimated time:** 90-120 minutes  
- **Outcome:** A working HR assistant with advanced orchestration patterns

---

## 1. Lab objectives

By the end of this lab, you will have built and tested:

1. A Copilot Studio agent (`HR Assist Agent`)
2. Skill-driven behavior
3. Trigger setup
4. Two hooks:
   - **Start (pre-loop)** hook -> workflow
   - **User prompt submitted** hook -> Python script
5. A case intake automation workflow
6. Agent-to-agent connectivity
7. Dataverse persistence with automatic updates when data changes

---

## 2. Prerequisites

You need:

1. Access to Copilot Studio (environment admin or maker permissions)
2. Permission to create/edit:
   - Agents
   - Workflows (Power Automate)
   - Dataverse tables/flows
3. Optional:
   - Teams and Outlook connectors for notifications
   - SharePoint for document/file actions

---

## 3. Lab architecture (what you are building)

1. User chats with **HR Assist Agent**
2. **Hook #1 (Session Start)** injects consistent safety/triage context
3. **Hook #2 (User Prompt Submitted)** executes Python risk detector
4. Agent can call workflow to create and escalate HR case
5. Agent can delegate payroll-specific requests to connected Payroll agent
6. Case data is stored in Dataverse and downstream updates run automatically

---

## 4. Task 1 - Create the base agent (10 min)

### Do this

1. Open Copilot Studio -> **Create agent**
2. Set name: `HR Assist Agent`
3. Open **Build** tab

### Add base instruction text

Paste this starter block (edit as needed):

```text
You are HR Assist Agent, a professional HR support assistant.
You handle:
1) Employee onboarding
2) Benefits/leave questions
3) Payroll FAQs
4) HR policy guidance
5) Employee records requests
6) Manager HR workflows

Rules:
- Be concise, empathetic, and professional.
- Confirm employee identity before taking any action.
- If policy details are unknown, do not invent; clearly state limitation and suggest HR Operations follow-up.
```

### Verify

- Agent appears as `HR Assist Agent`
- Instruction panel contains your text

### Screenshot reference

![Agent Build](images/01-agent-build.png)

---

## 5. Task 2 - Add skills and tools (10 min)

### Do this

1. In Build -> **Skills**, add/create:
   - `employee-onboarding`
   - `employee-records-request`
   - `manager-hr-workflows`
   - `hr-policy-qa`
2. In Build -> **Tools**, ensure at least:
   - Send email
   - Read SharePoint list items
   - Create a file in SharePoint

### Verify

- Skills list shows all skill entries
- Tools list shows required tools

---

## 6. Task 3 - Confirm trigger setup (5 min)

### Do this

1. In Build -> **Triggers**
2. Ensure `Chat with the agent` is present and active

### Verify

- Agent can be tested in Preview using chat

---

## 7. Task 4 - Add Hook #1 (workflow hook) (15 min)

## Goal

Inject deterministic context at session start.

### Do this

1. Build -> top-right **More options (...)** -> **Hooks (Preview)**
2. Click **Create a hook**
3. Set:
   - **Name:** `Session Start - Safety and Triage Context`
   - **When:** `Start (pre-loop)`
   - **Action:** `Run a workflow`
   - **Failure mode:** `Continue`
4. For workflow, create/select:
   - `Hook - SessionStart Safety and Triage`

### Implement workflow logic

In workflow designer:

1. Trigger: **When an agent calls the workflow**
2. Add/keep output field: `additionalContext`
3. Return a context string that includes:
   - intent classification categories
   - identity confirmation rule
   - sensitive-case handling rule
   - anti-hallucination policy rule

### Verify

- Hook appears in Hooks list
- Event type shows **Start (pre-loop)**
- Action shows **Run a workflow**

---

## 8. Task 5 - Add Hook #2 (Python hook) (20 min)

## Goal

Detect risky HR prompts before model response and adjust prompt safely.

### Do this

1. In **Hooks (Preview)** click **Create a hook**
2. Set:
   - **Name:** `Prompt Risk Detector (Python)`
   - **When:** `User prompt submitted`
   - **Action:** `Run a Python script`
   - **Failure mode:** `Continue`
3. Paste script:

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

### Verify

- Hook list now has 2 hooks
- Python hook row shows:
  - Event: `User prompt submitted`
  - Action: `Run a Python script`

### Screenshot references

![Hooks List](images/02-hooks-list.png)
![Add Hook Python](images/03-add-hook-python.png)

---

## 9. Task 6 - Add automation workflow (HR case intake) (15 min)

## Goal

Agent should be able to create a case record and trigger escalation notifications.

### Do this

Create a flow `HR Case Intake and Escalation` with:

1. Trigger: `When an agent calls the workflow`
2. Inputs:
   - `employeeId` (text)
   - `issueType` (text)
   - `summary` (text)
   - `severity` (text)
3. Actions:
   - Create record in Dataverse table (or SharePoint list for initial lab)
   - Send email/Teams notification to HR Ops
4. Respond to agent with:
   - `caseId`
   - `escalationStatus`
   - `createdAt`

### Workflow logic explained

- **Input capture** -> **Record creation** -> **Notification** -> **Response to agent**
- This makes chat actions operational and auditable.

### Verify

- Ask agent to create a case
- Receive a case ID in response

---

## 10. Task 7 - Agent-to-agent scenario (10 min)

## Goal

Delegate payroll specialist requests to another agent.

### Do this

1. Create second agent: `Payroll Specialist Agent`
2. In HR Assist Agent -> Build -> **Connected agents** -> Add payroll agent
3. Add delegation rule in instructions/skills:

```text
If request is payroll-specific (salary mismatch, tax deduction, arrears, bonus formula),
route to Payroll Specialist Agent.
```

### Verify

Test prompt:

`My salary and tax deduction look wrong this month.`

Expected:
- HR agent routes/uses connected payroll path

---

## 11. Task 8 - Dataverse storage + auto-update on change (15 min)

## Goal

Automatically run updates when case data changes.

### Do this

Create Dataverse table `hr_case`:

- `case_id`
- `employee_id`
- `category`
- `severity`
- `status` (New/In Review/Escalated/Closed)
- `owner`
- `last_updated`

Create cloud flow:

1. Trigger: **When a row is added, modified or deleted** (Dataverse)
2. Condition examples:
   - if `status = Escalated` -> notify HR leadership
   - if `status = Closed` -> notify requester and update summary metrics
3. Optional actions:
   - Update SLA table
   - Post Teams adaptive card

### Verify

1. Edit an existing case status in Dataverse (`New -> Escalated`)
2. Confirm automation flow runs and notification/update actions execute

---

## 12. Hands-on test sequence (run exactly in order)

## Test 1 - Baseline behavior

Prompt:
`How many vacation days do I have left?`

Expected:
- Normal answer style
- No high-risk safety mode

## Test 2 - Python hook trigger

Prompt:
`I think my manager is discriminating against me and I want to file a legal complaint.`

Expected:
- Safer, structured response behavior
- Empathy + no legal conclusion + escalation option

## Test 3 - Session-start workflow hook

Start a new chat and ask:
`What is the leave carry-forward policy?`

Expected:
- Consistent identity + safety guardrail behavior from first turn

## Test 4 - Case automation

Prompt:
`Create an HR case for employee E1234, issue type Payroll Dispute, severity High.`

Expected:
- Case created
- Case ID returned
- Notification sent

## Test 5 - Connected agent

Prompt:
`Explain why my tax deduction changed and calculate possible causes.`

Expected:
- Payroll-specialist pathway/delegation

## Test 6 - Dataverse auto update

Manual action:
- Change case status to `Escalated` in Dataverse

Expected:
- Update-trigger flow runs
- Dependent notifications/updates executed

---

## 13. Validation checklist (completion criteria)

Mark each as done:

- [ ] HR Assist Agent created
- [ ] Skills configured
- [ ] Trigger configured
- [ ] Hook #1 (Start -> Workflow) working
- [ ] Hook #2 (User prompt -> Python) working
- [ ] Case intake workflow returns case ID
- [ ] Connected agent delegation works
- [ ] Dataverse change-trigger automation works
- [ ] All six tests passed

---

## 14. Troubleshooting quick fixes

### Hook not firing

- Recheck **When** event
- Recheck **Action** type
- Save + Publish agent again

### Python hook no effect

- Ensure return uses `{"modifiedPrompt": ...}`
- Ensure script has `def main(event):`
- Keep failure mode as `Continue` while testing

### Workflow not returning outputs

- Confirm `Respond to the agent` node is configured
- Confirm output names match what caller expects
- Publish workflow after changes

### Dataverse flow not triggering

- Confirm row-change trigger is correct table
- Validate connector/permission
- Test with direct row edits first

---

## 15. Suggested next lab extensions

1. Add severity scoring in Python hook
2. Add human approval for High severity cases
3. Add monitor dashboards for hook/workflow execution
4. Add environment variables for mailbox/table/site names
