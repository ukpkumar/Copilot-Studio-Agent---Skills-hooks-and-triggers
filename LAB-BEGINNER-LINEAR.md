# Beginner Linear Lab: Build an HR Copilot Studio Agent (Skills + Triggers + Hooks + Workflow)

This is a **beginner-first, step-by-step** version with copy/paste blocks and screenshots.

---

## 1) What you will build

By the end, you will have:

- `HR Assist Agent`
- Skills + chat trigger
- Hook 1: session workflow guardrail
- Hook 2: Python per-message risk detector
- Case intake automation workflow

---

## 2) Prerequisites

1. Copilot Studio maker access  
2. Permission to create workflows and Dataverse tables  
3. 60-90 minutes

---

## 3) Step 1 - Create the agent

Create agent name:

```text
HR Assist Agent
```

Go to **Build** and paste this baseline instruction:

```text
You are HR Assist Agent, a professional HR support assistant.
Handle onboarding, records requests, manager workflows, policy questions, and payroll FAQs.

Rules:
- Be concise, empathetic, and professional.
- Confirm employee identity before taking action.
- Do not invent policy numbers, thresholds, or legal conclusions.
- Offer escalation when risk/sensitivity is high.
```

**You should now see:** Agent build page with instruction content.

![Agent Build](images/01-agent-build.png)

---

## 4) Step 2 - Add skills and tools

Add these skills:

- `employee-onboarding`
- `employee-records-request`
- `manager-hr-workflows`
- `hr-policy-qa`

Ensure tools include:

- Send email
- Read SharePoint list items
- Create a file in SharePoint

**You should now see:** skills + tools listed in Build panel.

---

## 5) Step 3 - Verify trigger

In **Triggers**, keep:

- `Chat with the agent`

**You should now see:** trigger active and Preview tab usable.

---

## 6) Step 4 - Add Hook 1 (Session workflow hook)

Open: **Build -> More options (...) -> Hooks (Preview)**  
Click **Create a hook** and set:

- **Name:** `Session Start - Safety and Triage Context`
- **When:** `Start (pre-loop)`
- **Action:** `Run a workflow`
- **Failure mode:** `Continue`

For workflow, create/select:

```text
Hook - SessionStart Safety and Triage
```

In that workflow, return `additionalContext` like this:

```text
Always verify employee identity before actions.
Do not provide legal conclusions.
Do not invent policy numbers if unknown.
Use sensitive-case handling for discrimination/harassment complaints.
```

**You should now see:** hook row with Start (pre-loop) + Run a workflow.

---

## 7) Step 5 - Add Hook 2 (Python hook)

In Hooks, click **Create a hook** and set:

- **Name:** `Prompt Risk Detector (Python)`
- **When:** `User prompt submitted`
- **Action:** `Run a Python script`
- **Failure mode:** `Continue`

Paste this script:

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

**You should now see:** second hook row with User prompt submitted + Run a Python script.

![Hooks List](images/02-hooks-list.png)
![Add Hook Python](images/03-add-hook-python.png)

---

## 8) Step 6 - Add case automation workflow

Create workflow:

```text
HR Case Intake and Escalation
```

Inputs:

- `employeeId`
- `issueType`
- `summary`
- `severity`

Actions:

1. Create case record (Dataverse/SharePoint)
2. Notify HR Ops (email/Teams)

Outputs back to agent:

- `caseId`
- `createdAt`
- `escalationStatus`

**You should now see:** case creation response with case ID.

---

## 9) Step 7 - Test sequence (copy/paste prompts)

### Test A - Safe

```text
How many vacation days do I have left?
```

Expected: normal response style.

### Test B - Risky

```text
I think my manager is discriminating against me and I want to file a legal complaint.
```

Expected: safer response style (more guarded, escalation-oriented).

### Test C - New chat baseline

Start new chat and send:

```text
What is the leave carry-forward policy?
```

Expected: session baseline guardrail behavior from first turn.

### Test D - Case automation

```text
Create an HR case for employee E1234, issue type Payroll Dispute, severity High.
```

Expected: case ID returned and notification action executed.

---

## 10) Quick explanation: why two hooks?

- **Session workflow hook:** one-time global setup for the whole chat
- **Python hook:** per-message dynamic check

Use both when you need:
1. stable session-wide policy baseline, and
2. message-level risk detection.

---

## 11) Troubleshooting quick fixes

- Hook not firing -> verify event/action + Save + Publish  
- Python no effect -> verify `modifiedPrompt` return  
- Workflow no output -> verify `Respond to the agent` outputs  
- Still failing -> republish agent after every hook/workflow change

