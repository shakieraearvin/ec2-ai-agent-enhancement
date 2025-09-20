# EC2 AI Agent Enhancement System

Netflix WL2 manual EC2 remediation extended with a conversational AI Agent. The agent identifies EC2 instance IDs from incident text, requests human approval, and executes the same restart through AWS Integration Server. Manual and AI paths share the same API and the same Remediation Log.



---

## 1. System Overview

This enhancement keeps the WL2 manual remediation fully intact and adds a chat interface.

- **Manual path**  
  Engineer opens an EC2 record or incident and clicks a UI Action. The system calls AWS Integration Server to restart the instance and writes to the Remediation Log.

- **AI path**  
  Engineer chats with the EC2 Remediation Assistant. The agent reads incident text to find an EC2 `instance_id`, asks for approval, calls the same AWS Integration Server, and writes to the same Remediation Log.

**What stays the same**
- AWS Integration Server connection and credentials  
- Remediation logic and logging table  
- EC2 monitoring data model

**Screenshots**  
Place images in `./screenshots/` and update links as needed.

---

## 2. Implementation Steps

### 2.1 Prerequisites
- WL2 manual remediation can restart an EC2 instance  
- EC2 Instance table has data  
- AWS Integration Server alias and credentials are valid  
- At least one incident includes an EC2 id in short description or description

### 2.2 Update Set activation
1. System Update Sets → Local Update Sets → New  
2. Name: `EC2 AI Agent Enhancement` → Submit → **Make Active**

### 2.3 AI Agent creation
- AI Agent Studio → Agents → New  
  - Name: **EC2 Remediation Assistant**  
  - Description: Helps DevOps engineers restart EC2 instances from incidents by reading descriptions and executing remediation with human approval  
  - Enable **Run from a record** for **Incident**  
  - Limit access to a DevOps test group

**Agent instructions summary**
1) Extract EC2 instance IDs from user text or incident text.  
2) If on an Incident record, first call the lookup tool by Incident Number, then extract the ID from returned text.  
3) If multiple IDs appear, ask which to use.  
4) Ask for explicit approval before execution.  
5) On approval, call the restart tool.  
6) Report success or failure and the Remediation Log number.

### 2.4 Tools

**A. Record Operation: Short Description Look Up**  
- Type: Record operation  
- Table: Incident  
- Operation: Look up records  
- Input: `incident_number`  
- Condition: Number is `{{incident_number}}`  
- Returned fields: Short description, Description, Number  
- Max records: 1  
- Execution mode: Unsupervised  
- Purpose: Provide incident text so the agent can extract an EC2 id

**B. Script Tool: Restart EC2 Instance**  
- Type: Script  
- Execution mode: Supervised  
- Input expected by the tool: `instance_id` (example `i-09ae69f1cb71f622e`)  
- Output returned by the tool: success status, message, remediation log id, HTTP status, response time  
- Purpose: Perform the restart via AWS Integration Server and write the Remediation Log

**Tool linking**  
- Agent → Tools tab → add **Short Description Look Up**  
- Agent → Tools tab → add **Restart EC2 Instance**

### 2.5 Supervision and approval
- Lookup is read only and unsupervised  
- Restart is supervised and requires Approve in chat  
- Supervised runs create execution plan and task records for audit

### 2.6 Logging parity
- Both paths write to `x_snc_ec2_monito_0_remediation_log`  
- Typical fields: `ec2_instance`, `attempted_status`, `timestamp`, `request_payload`, `success`, `response_payload`, `http_status_code`, `response_time_ms`, and `error_message` when present

---

## 3. Architecture Diagram

Add **Diagram.png** at the repo root and reference it here:

![Enhanced Architecture](./Diagram.png)

**Show**
- **Manual workflow**  
  Incident or EC2 record → UI Action → Remediation logic → AWS Integration Server → AWS EC2 → Remediation Log

- **AI workflow**  
  Chat → AI Agent → Incident lookup by Number → extract instance id → Approve → Restart Script Tool (supervised) → AWS Integration Server → AWS EC2 → Remediation Log

- **Decision points**  
  When you are already on the record with one instance, manual is fine.  
  When multiple instances fail or you want chat speed, use the AI Agent.  
  Both paths share the same AWS Integration Server and the same logging table.

---

## 4. Optimization

### 4.1 When to use which method

| Scenario | Manual UI Action | AI Agent chat |
|---|---|---|
| Already on an EC2 record | Best | Good |
| Working from an incident with an id in text | Good | Best |
| Multiple failures during a surge | OK | Best |
| Inline human approval and audit | Good | Best |

### 4.2 Efficiency
- AI removes navigation to each EC2 record during bursts.  
- Lookup returns text quickly, the agent extracts the id, and approval happens in chat.  
- Both write the same logs, so reporting and audit stay consistent.

### 4.3 Trade offs
- Separate read only lookup from supervised execution for safety.  
- Supervised runs add execution plan records that must be exported.

---

## 5. DevOps Usage

### Manual remediation
1) Open the EC2 instance record.  
2) Click the restart UI Action.  
3) Confirm the Remediation Log entry and AWS response.

### AI conversational remediation

**From an Incident**
1) Open the incident that contains an EC2 id in short description or description.  
2) Launch EC2 Remediation Assistant.  
3) Say: “Help me with this incident.”  
4) Agent calls lookup with the incident number, extracts the id, and asks: “Approve or Cancel.”  
5) Type “Approve.”  
6) Agent restarts the instance and returns the Remediation Log number.

**From anywhere**
1) Open chat with EC2 Remediation Assistant.  
2) Say: “Restart i-09ae69f1cb71f622e.”  
3) On “Approve,” agent restarts and returns the Remediation Log number.

**Common errors**
- Invalid id format → agent asks for a corrected id.  
- No id in text → agent asks for an instance id.  
- API failure → agent shows HTTP code and error details and the log contains payloads.

---

## Screenshots (with presenter notes)

> Place all images in `./screenshots/` using the filenames below. Each image includes short notes you can read while presenting.

---

### 01 – Agent record
![01 – Agent record](./screenshots/01-agent-record.png)

**What you are showing:** The EC2 Remediation Assistant is configured to run from Incident records and is scoped to DevOps users.  
**What to say (10–15 sec):** “This is the AI agent. It can launch from an Incident, read the text on the record, and it always requires human approval before it does anything.”  
**Where in UI:** AI Agent Studio → Agents → EC2 Remediation Assistant.

---

### 02 – Agent Tools tab
![02 – Agent Tools tab](./screenshots/02-agent-tools-tab.png)

**What you are showing:** Both tools are linked to the agent: the read only lookup and the supervised restart tool.  
**What to say:** “The agent first calls a read only lookup for the incident text, then on approval it calls the restart tool. Two tools keep safety clear.”  
**Where in UI:** AI Agent Studio → Agents → EC2 Remediation Assistant → Tools tab.

---

### 03 – Lookup tool config (Record Operation)
![03 – Lookup tool config (Record Operation)](./screenshots/03-lookup-record-op-config.png)

**What you are showing:** Record Operation set to Incident, Look up records by Incident Number, return Short description and Description.  
**What to say:** “This lookup is unsupervised and read only. It gives the agent the text so it can extract the EC2 instance id.”  
**Where in UI:** AI Agent Studio → Tools → your lookup tool.

---

### 04 – Lookup test result
![04 – Lookup test result](./screenshots/04-lookup-test-result.png)

**What you are showing:** Example result that includes Short description and Description containing an EC2 id.  
**What to say:** “Here is the text the agent reads. It finds the id pattern i-xxxxxxxx and gets ready to ask for approval.”  
**Where in UI:** AI Agent Studio → Tools → your lookup tool → Test.

---

### 05 – Restart tool config (Script)
![05 – Restart tool config (Script)](./screenshots/05-restart-tool-config.png)

**What you are showing:** Script Tool is set to Supervised and expects `instance_id` as input.  
**What to say:** “This tool is supervised. The agent cannot run it until a human types Approve.”  
**Where in UI:** AI Agent Studio → Tools → Restart EC2 Instance.

---

### 06 – Restart tool script
![06 – Restart tool script](./screenshots/06-restart-tool-script.png)

**What you are showing:** The provided restart script that calls the AWS Integration Server and writes to the Remediation Log.  
**What to say:** “This is the same remediation logic as manual WL2. Same API, same logging table, just triggered through chat.”  
**Where in UI:** Same tool → Script section.

---

### 07 – Supervised approval dialog
![07 – Supervised approval dialog](./screenshots/07-approval-dialog.png)

**What you are showing:** The Approve or Cancel step with the `instance_id` visible before execution.  
**What to say:** “Human in the loop. The tool only runs after explicit Approve, which gives us audit and safety.”  
**Where in UI:** AI Agent Studio → Test the agent and trigger the restart flow.

---

### 08 – Execution plan
![08 – Execution plan](./screenshots/08-execution-plan.png)

**What you are showing:** The execution plan record created by the supervised run.  
**What to say:** “Supervised runs generate execution plans for traceability.”  
**Where in UI:** System Definition → Tables → search `sn_aia_execution_plan` → open the plan created by your run.

---

### 09 – Execution plan tasks
![09 – Execution plan tasks](./screenshots/09-execution-plan-tasks.png)

**What you are showing:** All tasks linked to the execution plan.  
**What to say:** “These tasks show each supervised step, which we include in the update set.”  
**Where in UI:** Open the execution plan → click the Completed State link to view tasks.

---

### 10 – Remediation Log entry (AI path)
![10 – Remediation Log entry (AI path)](./screenshots/10-remediation-log.png)

**What you are showing:** New log row created by the agent run, with the same fields as manual WL2.  
**What to say:** “Same Remediation Log as manual. This proves parity for auditing and reporting.”  
**Where in UI:** Remediation Log table in your scoped app.

---

### 11 – Manual UI Action (baseline)
![11 – Manual UI Action (baseline)](./screenshots/11-manual-ui-action.png)

**What you are showing:** The original WL2 manual restart on an EC2 record.  
**What to say:** “Baseline path still works. We keep the manual UI for single cases.”  
**Where in UI:** EC2 Instance record → your WL2 restart UI Action.

---

### 12 – AI chat success (AI path)
![12 – AI chat success (AI path)](./screenshots/12-ai-chat-success.png)

**What you are showing:** The conversation flow: extract id from incident text, ask Approve or Cancel, run, then return the Remediation Log number.  
**What to say:** “During peak hours the chat path is faster. It extracts the id, gets approval, and triggers the same restart.”  
**Where in UI:** AI Agent Studio → Test → run the end to end chat.

---

## Script comparison analysis

**Inputs**  
- Manual uses the current record context.  
- AI uses an `instance_id` extracted from incident text or provided by the user.

**Lookup**  
- Manual often acts on the current EC2 record.  
- AI queries the incident by Number, reads text, and extracts the EC2 id.

**Execution**  
- Both call the same AWS Integration Server endpoint for restart.

**Returns**  
- Manual provides UI feedback and a log entry.  
- AI provides chat feedback and the same log entry reference.

**Error handling**  
- Both validate the instance id format, handle not found, and surface API errors.

**When to use**  
- Manual for single, focused incidents when you are already on the record.  
- AI for peak periods and chat-first workflows with built-in approval.

---

## Update set contents and export

**Include**
- Agent definition record `sn_aia_agent`  
- Both tool records  
- Agent to tool links `sn_aia_agent_tool_m2m`  
- Execution plan and all related tasks from a supervised run  
- Connection alias records if required by reviewers

**Export**
- System Update Sets → Local Update Sets → open `EC2 AI Agent Enhancement` → Export to XML  
- Save as **ec2-ai-agent-enhancement.xml** at repo root

---

### Notes for reviewers
- All executable configuration, scripts, and relationships are packaged in `ec2-ai-agent-enhancement.xml`.  
- Screenshots in `./screenshots/` illustrate agent settings, tools, supervised audit records, and logs.
