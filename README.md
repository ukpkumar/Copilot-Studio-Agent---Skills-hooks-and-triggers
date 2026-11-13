# From Chat to Action: A Hands-On Copilot Studio HR Agent Lab

Build an HR assistant with **instruction-based skills, chat triggers, workflow and Python hooks, Dataverse automation, and a connected Payroll Specialist Agent**.

**Start here: [Open the complete beginner hands-on guide](LAB-BEGINNER-LINEAR.md).**

**Optional deep dive: [Hooks explained: all events, fields, scenarios, and Workflow versus Python](HOOKS-REFERENCE.md).** Includes the same prompt-processing requirement implemented both ways, pre-tool decisions, failure modes, and runtime evidence.

The guide provides click-by-click instructions, complete skill descriptions and procedures, copy/paste agent prompts and Python code, connector configuration, flow field mappings, test prompts, expected results, and production preparation.

## Choose your reading route

**Linear route:** Read the [beginner guide](LAB-BEGINNER-LINEAR.md) from Step 1 through Step 14.

**Module route:** Use the index below to open the corresponding detailed exercise. Both routes use the same complete instructions, so there is no second abbreviated "add a skill" checklist to guess from.

| Module | What you will actually build | Detailed exercise |
|---|---|---|
| 1. Environment and agent | Test environment, demo values, agent creation prompt, baseline instructions | [Prepare](LAB-BEGINNER-LINEAR.md#step-1-prepare-your-environment-and-demo-values) and [create the agent](LAB-BEGINNER-LINEAR.md#step-2-create-the-agent-and-paste-its-instructions) |
| 2. Four skills | Name, description, full instructions, and a test for each HR skill | [Create skills](LAB-BEGINNER-LINEAR.md#step-3-create-four-skills-with-complete-instructions) |
| 3. Entry points | Distinguish Copilot/Teams chat triggers from Preview authoring tests | [Configure the trigger](LAB-BEGINNER-LINEAR.md#step-4-understand-and-configure-the-chat-trigger) |
| 4. Tools | Outlook and SharePoint action selection, connections, fixed destinations, approval instructions | [Configure connector tools](LAB-BEGINNER-LINEAR.md#step-5-configure-connector-tools) |
| 5. Workflow hook | Hook-generated event contract and a populated additionalContext response | [Build the context hook](LAB-BEGINNER-LINEAR.md#step-6-create-the-session-context-workflow-hook) |
| 6. Python hook | Complete prompt-processing function with matching/nonmatching behavior and limitations | [Build the Python hook](LAB-BEGINNER-LINEAR.md#step-7-create-the-prompt-processing-python-hook) |
| 7. Hook comparison | Concrete examples, workflow output inspection, Python transformation demonstration | [See the difference](LAB-BEGINNER-LINEAR.md#step-8-see-the-difference-between-the-two-hooks) |
| 8. Storage | Solution, table, column types, logical names, and a manual storage test | [Create Dataverse storage](LAB-BEGINNER-LINEAR.md#step-9-create-the-dataverse-case-table) |
| 9. Case automation | Trigger inputs, validation, row mappings, response outputs, and attaching the flow as an agent tool | [Build case intake](LAB-BEGINNER-LINEAR.md#step-10-build-and-attach-the-case-intake-automation) |
| 10. Event automation | Modified-row trigger, filtering attributes, status branches, loop prevention, optional test email | [Process status changes](LAB-BEGINNER-LINEAR.md#step-11-automatically-update-records-when-status-changes) |
| 11. Connected agent | Full specialist instructions, publishing prerequisite, delegation description, arithmetic test | [Connect Payroll Specialist](LAB-BEGINNER-LINEAR.md#step-12-create-and-connect-a-payroll-specialist-agent) |
| 12. Demonstration and production preparation | Twelve ordered tests, completion criteria, safe cleanup, production preparation | [Run the demonstration](LAB-BEGINNER-LINEAR.md#step-13-run-the-complete-demonstration) and [prepare for production](LAB-BEGINNER-LINEAR.md#step-14-clean-up-and-prepare-for-production) |

## Important corrections about hooks

**Event and implementation are independent.** The event determines *when* a hook runs; Workflow or Python determines *how* it runs. A workflow is not inherently session-scoped, and Python is not inherently per-message.

The lab deliberately selects Start (pre-loop) for context and User prompt submitted for prompt processing. Start's observed UI description says "once before the reasoning loop begins"; do not promise one run per chat across preview versions without runtime evidence.

The keyword decision in Python is deterministic, but the model's response is not. Prefixes and additional context are not authorization controls. A debug marker in model input also does not automatically appear in chat output.

**You do not need both hooks just to repeat the same HR safety instructions.** See the [examples and evidence exercise](LAB-BEGINNER-LINEAR.md#step-8-see-the-difference-between-the-two-hooks) for when each pattern is useful.

## Scope, screenshots, and implementation status

This is a learning guide, not a deployable solution export. The original demonstration agent had the two hooks configured. Additional case automation, Dataverse, and specialist-agent exercises must be built by the learner; this repository does not claim they have already been deployed.

New screenshots show the actual skill editor, trigger picker, and tool picker. Older hook screenshots are explicitly labeled historical illustrations in the guide; follow the written event/action settings rather than copying a starter dialog blindly.

The guide targets the next-generation Copilot Studio preview experience. Feature availability, connectors, licensing, and labels can differ by tenant. Use fictional data and a development environment.
