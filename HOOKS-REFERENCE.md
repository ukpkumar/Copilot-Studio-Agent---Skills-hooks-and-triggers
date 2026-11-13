# Copilot Studio Hooks: Events, Actions, and Practical Scenarios

[Back to the lab](LAB-BEGINNER-LINEAR.md) | [Repository index](README.md)

**Optional reading:** You can finish the main lab without implementing these extra scenarios. Use this page when choosing a hook or explaining it to a customer.

## Index

1. [Two choices: when and how](#1-two-choices-when-and-how)
2. [Every field in Add hook](#2-every-field-in-add-hook)
3. [All six event options](#3-all-six-event-options)
4. [Run a workflow versus Run a Python script](#4-run-a-workflow-versus-run-a-python-script)
5. [Same event, same requirement, two implementations](#5-same-event-same-requirement-two-implementations)
6. [Workflow-only connected-system example](#6-workflow-only-connected-system-example)
7. [Pre-tool decision example](#7-pre-tool-decision-example)
8. [Failure mode and recovery](#8-failure-mode-and-recovery)
9. [Testing and visual evidence](#9-testing-and-visual-evidence)
10. [Reuse, publishing, and production limits](#10-reuse-publishing-and-production-limits)
11. [Sources and preview differences](#11-sources-and-preview-differences)

## 1. Two choices: when and how

A hook automatically executes logic at a selected point in the agent lifecycle.

```text
WHEN the selected event happens
  RUN the selected action
  READ the event-compatible response
  CONTINUE according to that response and the hook's failure behavior
```

**When** answers: "At what moment should this happen?"

**Action** answers: "Should I implement that logic using a workflow or Python?"

| Selection | Example | Meaning |
|---|---|---|
| When | Start (pre-loop) | Supply context at the conversation-start lifecycle point |
| When | User prompt submitted | Process each submitted message |
| Action | Run a workflow | Execute the selected compatible workflow |
| Action | Run a Python script | Execute the entered Python code in an isolated sandbox |

**These are independent dimensions.** You can choose a workflow for prompt processing. You can choose Python for Start context if your editor exposes that event/action combination and its response contract.

It is incorrect to teach "workflow = session" and "Python = per-message" as a product rule. Those were merely the two combinations selected in the original demonstration.

### Do I actually need both original hooks?

Not to repeat the same discrimination guidance. Their purposes overlap:

1. The original Start workflow supplies general HR safety context.
2. The original prompt Python script detects a small set of keywords and adds similar guidance.

A clearer customer demonstration is **workflow to retrieve approved context** versus **Python to normalize or validate local data**. If the context never changes and no orchestration is needed, ordinary agent instructions may be simpler than a Start hook.

## 2. Every field in Add hook

Open the agent's top command bar **More options (...) > Hooks > Create**.

| Field/control | What to enter or select | Why it matters |
|---|---|---|
| Name | A descriptive name such as `Check demo email destination before send` | Identification only; it is not an instruction or trigger keyword |
| When / Event type | One of the six lifecycle events below | Determines available input/output fields |
| Action | Run a workflow or Run a Python script, where offered | Chooses implementation, not lifecycle timing |
| Tools | Specific target tools for tool-related events, when the field is available | Limits which tool calls raise this hook; empty means all tools in the documented experience |
| Workflow | A compatible published workflow | Compatibility depends on event input/output names and types |
| New workflow | Create from the selected event's template | Supplies the expected trigger and response contract |
| Python script | A complete `def main(event):` implementation | Reads an event dictionary and returns an event-compatible decision dictionary |
| Failure mode | Whether the agent continues or stops if this hook errors/times out | This is about failure of the hook itself, not a successful deny decision |
| Done | Apply changes to the draft agent | Does not replace Save or Publish |
| Cancel / Close | Leave the editor, discarding changes only after confirmation if prompted | Do not accidentally discard unrelated agent edits |

### Common event input and metadata

The documented workflow contract always includes:

| Input | Meaning |
|---|---|
| `event` | Lifecycle event identifier |
| `timestamp` | When that event was raised |
| `metadata` | Conversation, user, channel, and agent context |

Metadata can include `conversationId`, `userId`, `userAadObjectId`, `userDisplayName`, `channel`, `agentId`, `agentName`, and `agentSchemaName`.

Fields can be empty, especially without sign-in. A channel user ID is not interchangeable with an Entra object ID. Validate identity and permissions before using a connector; do not treat a user-supplied employee ID as authentication.

For Python, read the input/return-field help directly below the editor after selecting the event. Do not copy one event's return fields into another.

## 3. All six event options

The following contracts describe the linked Microsoft Learn workflow reference. Use the generated template and your current editor as the final check for your preview version.

### 3.1 Start (pre-loop)

**Use for:** Context needed before conversation reasoning starts.

**Documented lifecycle:** A new conversation begins. The UI describes it as running once before the reasoning loop begins. Validate actual Preview traces if demonstrating execution counts.

**Extra inputs:** None beyond event, timestamp, metadata.

**Return:** `additionalContext`.

**Scenario:** An employee starts a chat. A workflow looks up an approved region-to-policy mapping using authorized identity metadata, then supplies the relevant policy reference. The agent need not ask for the region every time.

**Simple lab response**

```json
{
  "additionalContext": "Fictional HR lab. Context version HRLAB-CONTEXT-V1. Use owner roles Manager, IT, and HR Operations."
}
```

**Test:** Start a new chat, request the active lab context version, and inspect the workflow's actual response. Starting another conversation should raise a new Start event under the documented lifecycle.

**Do not use it for:** Creating a case every time someone opens chat. A lifecycle event is not user approval for a business write.

### 3.2 User prompt submitted

**Use for:** Standardizing the current message before the agent processes it.

**Extra input:** `prompt`, which is untrusted user input.

**Returns:** `modifiedPrompt`, `suppressOutput`.

**Scenario:** Recognize the lab shortcut `HRHELP` and replace it with a complete HR-help request. Both workflow and Python implementations are provided in Section 5.

**Example**

```json
{
  "modifiedPrompt": "Explain the HR lab assistant's capabilities. Do not save or send anything."
}
```

**Test:** Compare `HRHELP` with an ordinary onboarding prompt; inspect returned data rather than relying solely on tone.

**Important:** `suppressOutput` hides output; it is not a guarantee that a write operation cannot happen. A prompt prefix does not become an enforceable tool permission.

### 3.3 Error

**Use for:** Handling an agent error outside a tool call.

**Extra inputs:** `error`, `errorContext`, `recoverable`.

**Returns:** `errorHandling` (`retry`, `skip`, or `abort`), `retryCount`, `userNotification`, `suppressOutput`.

**Scenario:** A recoverable reasoning-stage interruption occurs. The hook chooses a bounded recovery response or a clear stop message rather than exposing diagnostics.

**Illustrative response**

```json
{
  "errorHandling": "abort",
  "userNotification": "The assistant could not complete this request. Please try a new request."
}
```

**Test:** Use a controlled test failure supported by your development environment, or test the handler using synthetic event inputs. A synthetic test does not prove that the live agent raised Error.

**Cautions:** Treat error text as untrusted. Do not print stack traces, tokens, private data, or connection details to users. `recoverable` is advisory; it does not prove that replay is safe.

### 3.4 Pre tool use

**Use for:** Checking a planned tool call before its action occurs.

**Extra inputs:** `toolName`, `parameters`.

**Returns:** `permissionDecision`, `permissionDecisionReason`, `modifiedParameters`, `additionalContext`, `suppressOutput`.

**Documented blocking decision:** `permissionDecision: "deny"`. The cited reference says an empty decision allows default processing.

**Scenario:** Before a demo email tool runs, reject a recipient outside your fixed test allow-list.

**Example response**

```json
{
  "permissionDecision": "deny",
  "permissionDecisionReason": "This lab permits email only to the configured test mailbox."
}
```

**Test:** Use a harmless mock tool or a synthetic event first. For an end-to-end send test, use only a mailbox you own and inspect that denied calls never reach the send action.

**UI difference:** The UI description says this event can allow, deny, or ask for confirmation. The cited Learn contract explicitly documents `deny` and default allow, but does not establish a confirmation enum. Do not guess `ask`, `confirm`, or another value. Use the generated contract/current help if it exposes confirmation, or use the tool's supported confirmation setting.

**Boundary:** This is the documented event for rejecting a planned tool action. Post-tool handling is too late to prevent that action. Connector permissions and server-side authorization are still required.

### 3.5 Post tool use

**Use for:** Processing a tool result or recording its outcome after execution.

**Extra inputs:** `toolName`, `parameters`, `result`.

**Returns:** `modifiedResult`, `additionalContext`, `suppressOutput`.

**Scenario:** A demo directory lookup returns fields the model does not need. Return only Department and EmployeeDemoId to the model; a separate workflow could record a minimal audit event if approved.

**Conceptual transformation**

```text
Tool result: employee demo ID + department + extra fields
Hook result: employee demo ID + department only
```

**Test:** With synthetic result data, verify excluded fields are absent from the returned `modifiedResult`. Then inspect runtime traces using fictional records.

**Preview difference:** The observed UI says this event runs after a tool finishes whether successful or failed. The cited Learn article describes successful completion. Do not build exactly-once audit logic on an assumed relationship with After tool failure; inspect what your environment raises.

**Limit:** Redacting what the model sees does not erase the original result from connector logs, prior traces, storage, or already delivered messages.

### 3.6 After tool failure

**Use for:** Giving the model accurate guidance when a tool returns an error.

**Extra inputs:** `toolName`, `parameters`, `error`.

**Return:** `additionalContext`.

**Scenario:** A case-create tool times out. Tell the agent not to invent a case ID and not to blindly repeat a possibly completed write.

**Example**

```json
{
  "additionalContext": "The tool failed or timed out. Do not claim a case was created or invent a case ID. The write outcome may be uncertain; inspect the case table and run history before retrying."
}
```

**Test:** Use a development mock that returns a known error, or synthetic handler inputs. Inspect the returned guidance and confirm there is no success-shaped case response.

**Difference from Error:** Error handles failures outside a tool call. After tool failure receives the tool's name, arguments, and error. The latter's documented contract does not include `errorHandling` or `retryCount`.

## 4. Run a workflow versus Run a Python script

| Question | Run a workflow | Run a Python script |
|---|---|---|
| How is it authored? | Designer nodes, conditions, inputs, outputs | Python function `main(event)` |
| What starts it? | The selected lifecycle event | The selected lifecycle event |
| Can it run on each prompt? | Yes, when configured for User prompt submitted | Yes, when configured for User prompt submitted |
| Can it supply Start context? | Yes, using the Start contract | If the current script editor offers that event/contract |
| External information? | Can orchestrate permitted connector/system actions available in the workflow environment | Observed sandbox explicitly has no network, connectors, or credentials |
| Good fit | Approved data lookups, connector orchestration, branching across systems | Local normalization, parsing, fixed-rule matching, compact validation |
| Deployment | Publish workflow, bind hook, save/publish parent agent | Save script in hook, save/publish parent agent |
| Reuse | A published workflow reference can be shared by multiple hooks/agents | Do not assume shared script-library behavior; maintain reviewed code consistently |
| Debugging | Workflow inputs, node outputs, run history; agent trace where available | Editor help and runtime details where exposed; local tests for pure logic |
| Operational tradeoff | Connection governance, additional actions, latency/capacity considerations | Code review, input validation, sandbox/runtime/time limits |
| Guarantees? | Consistent invocation at the event does not guarantee success | Deterministic code decisions do not make model replies deterministic |

**Choose workflow** when the rule needs an authorized lookup or system operation.

**Choose Python** when the rule can be computed completely from the event payload using supported sandbox capabilities.

Neither is automatically faster, cheaper, safer, or more maintainable for every requirement. Measure the actual scenario. Do not use Python to attempt to bypass connection governance.

## 5. Same event, same requirement, two implementations

This removes the confusion caused by comparing different events and implementations at once.

**Requirement:** When a user submits exactly `HRHELP`, replace it with a full help request. Leave every other message unchanged.

**Event for both:** User prompt submitted.

### Option A: Run a workflow

1. Open Hooks > Create.
2. Name it `Expand HRHELP - Workflow`.
3. Set When **User prompt submitted**, Action **Run a workflow**.
4. Select **New workflow**.
5. Preserve the generated event inputs and response output names/types.
6. Add a Condition before the response.
7. On the left side use an expression that lowercases/trims the **prompt input token**; compare it with literal `hrhelp`. Use the designer's input token rather than guessing an internal trigger property path.
8. In the matching branch, return this Text value in `modifiedPrompt`:

```text
Explain the HR lab assistant's capabilities. Do not save or send anything.
```

9. In the other branch, return the original prompt input as `modifiedPrompt`.
10. Keep `suppressOutput` false/default in both branches. Keep both response schemas consistent.
11. Save/publish the workflow, return to the hook dialog, select Done, and save the agent.

### Option B: Run a Python script

1. Create a separate hook named `Expand HRHELP - Python`.
2. Choose the **same When: User prompt submitted**.
3. Select **Action: Run a Python script**.
4. Paste:

```python
def main(event):
    prompt = event.get("prompt", "")
    if not isinstance(prompt, str):
        raise TypeError("Expected event.prompt to be a string")
    if prompt.strip().casefold() == "hrhelp":
        return {
            "modifiedPrompt": (
                "Explain the HR lab assistant's capabilities. "
                "Do not save or send anything."
            )
        }
    return {}
```

5. Choose Continue for this noncritical demo, select Done, and save the agent.

### Compare them fairly

Use **only one version at a time** in a disposable lab agent. Use a supported disable control if present, or record the configuration and remove/re-add it as needed. Do not keep both active and assume an undocumented processing order.

| Submitted text | Expected effective prompt |
|---|---|
| `HRHELP` | Expanded help request |
| ` hrhelp ` | Expanded help request |
| `Draft onboarding tasks.` | Original message |
| `What does HRHELP mean?` | Original message; exact shortcut rule does not match |

**Conclusion:** They implement the same rule at the same event. Python is compact for this purely local operation; workflow is useful if the next requirement adds a permitted system lookup. You do not need both.

## 6. Workflow-only connected-system example

**Scenario:** A Start hook gets approved regional HR guidance from a Dataverse configuration table.

This is an optional design exercise, not already deployed in the sample agent.

1. In your test environment, create a configuration table with Region Code, Policy Reference, and Guidance Text.
2. Add a fictional UAE lab row with a clearly labeled sample policy reference; do not invent a real company policy.
3. Determine the user's region through an approved identity-to-region mapping. Do not trust an employee ID supplied in the prompt.
4. Create a Start hook and use New workflow to obtain its contract.
5. Add authorized lookup steps using the signed-in identity metadata only when present and permitted.
6. Validate that the lookup returns one applicable configuration record. If identity/mapping is missing, return explicit context saying the region is unknown and should be requested; do not silently substitute a region.
7. Return approved minimal guidance in `additionalContext`.
8. Test known identity, unknown identity, and denied-connection cases separately.
9. Inspect workflow outputs without publishing identity metadata or private results.

**Why workflow here?** It can use an approved connection to retrieve configuration.

**Why not the observed Python sandbox?** It cannot call Dataverse or Microsoft Graph, because it has no network/connector credentials. Hard-coding the region in Python is not equivalent to an authorized lookup.

## 7. Pre-tool decision example

**Scenario:** A purely local rule rejects an email recipient outside a fixed demo allow-list.

### Workflow approach

1. Choose When **Pre tool use** and scope Tools to the demo email tool.
2. Create the event-compatible workflow.
3. Inspect the actual `parameters` shape in a harmless development test.
4. Extract and normalize every recipient field used by the tool, including CC/BCC if permitted. Reject unrecognized shapes rather than assuming a recipient path.
5. If any recipient is outside the configured test allow-list, return `permissionDecision = deny` and a short reason.
6. Otherwise leave the permission decision empty for the documented default behavior.
7. Use a supported confirmation mechanism for approved sends; allow-list membership does not establish user consent.

### Python approach

The same local decision is a possible Python use case **if your Pre tool script editor exposes the documented inputs and deny output**. Build against the actual parameters schema and editor contract.

Do not paste a generic `parameters["to"]` script into a production Outlook tool: tools can use different field names, nested objects, recipient arrays, or multiple recipient fields. A sample for the wrong schema is not an effective guardrail.

**Teaching point:** Python can calculate a decision from local event data. Workflow can calculate the same decision and can also retrieve a centrally managed policy using a permitted connector.

## 8. Failure mode and recovery

There are three different situations:

| Situation | Example | What controls it |
|---|---|---|
| Hook successfully rejects an action | Pre-tool response says deny | Event decision contract |
| Tool itself fails | Dataverse action times out | After tool failure event; ordinary tool error handling |
| Hook itself fails | Python exception or workflow timeout | Hook Failure mode where offered |

### Continue

The agent continues when the hook fails or times out. Appropriate for a noncritical demo shortcut or optional context enrichment.

**Risk:** A failed policy-check hook can leave the action without that check. Never treat Continue as fail-closed enforcement.

### Stop behavior, where offered

Your screenshot's helper text says the setting chooses whether the agent continues or stops on hook error/timeout. Select the stopping option exposed by your current dropdown when you intend failure to halt processing.

Do not infer the exact option label, response message, or tenant-wide enforcement guarantees from an older article. Test the behavior in a disposable agent before relying on it.

**Tradeoff:** An unavailable dependency can interrupt user work. Enforce business-critical permissions in the backend as well.

### Controlled failure demonstration

1. Use a disposable lab agent and a non-writing event.
2. Temporarily replace the script with:

```python
def main(event):
    raise RuntimeError("Intentional fictional lab hook failure")
```

3. Save, submit a harmless message, and observe Continue behavior.
4. If a stopping mode exists, select it, save, and repeat.
5. Record the actual result and traces; do not invent a success claim.
6. Restore the working script before any demonstration or publication.

This deliberately tests hook failure, not keyword matching and not a tool failure.

## 9. Testing and visual evidence

| What you want to prove | Stronger evidence | What is insufficient |
|---|---|---|
| Hook configuration | Saved event/action and bound workflow/script | A name in a screenshot |
| Workflow execution | Corresponding run inputs/outputs and timestamp | Agent says "I used a hook" |
| Prompt transformation | Runtime `modifiedPrompt` if exposed | More empathetic wording |
| Pure Python logic | Local function outputs for synthetic events | Claim that local execution proves cloud invocation |
| Denied operation | Deny decision plus absence of tool execution | A refusal after the email was sent |
| Failure handling | Intentional safe failure with recorded failure mode | Assuming no visible error means success |

See the main lab's [hook evidence exercise](LAB-BEGINNER-LINEAR.md#step-8-see-the-difference-between-the-two-hooks).

Do not promise a specific Monitor/trace panel exposes every script payload. If unavailable, label that evidence gap. Model context markers do not automatically become visible chat messages.

## 10. Reuse, publishing, and production limits

1. Save/publish a workflow before binding it for runtime use.
2. Done changes the draft hook; save the parent agent to persist it.
3. Publish the parent agent only after testing and distribution review.
4. Changing a shared workflow affects hooks that reference it; it is not a per-agent copy.
5. Changing event type can clear the workflow selection because the old contract may not match.
6. Scope tool hooks narrowly before expanding to all tools.
7. Do not assume ordering, transactional behavior, automatic rollback, recursive-hook behavior, or exactly-once execution without documentation/runtime evidence.
8. Treat prompts, tool parameters/results, and diagnostic errors as untrusted data.
9. Avoid copying personal data into audit logs; use approved retention and permissions.
10. Use idempotency before retrying writes. A timeout does not prove that the operation never happened.

## 11. Sources and preview differences

Primary source: [Microsoft Learn: Hooks (Microsoft Copilot) (preview)](https://learn.microsoft.com/en-us/microsoft-copilot-studio/agents-experience/hooks-overview).

This page was prepared against the source available on **14 November 2026** and the supplied/observed preview interface.

There are real differences:

| Area | Source article | Observed preview UI |
|---|---|---|
| Action types | Says action is always a workflow | Offers Run a workflow and Run a Python script |
| Hook failure | Describes continuing on failure | Exposes a Failure mode setting with continue/stop helper text |
| Post-tool lifecycle | Describes successful tool completion | Description says succeeded or failed |
| Pre-tool confirmation | Explicitly documents deny/default allow | Description also mentions asking for confirmation |

Accordingly, do not claim that every response field/value is verified for both action types in every tenant. For implementation, select the event, read its live editor/template contract, preserve required names/types, and test that particular configuration.
