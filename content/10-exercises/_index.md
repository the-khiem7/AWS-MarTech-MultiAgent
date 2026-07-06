---
title: "Hands-On Exercises"
date: 2026-07-06
weight: 9
chapter: false
pre: " <b> 9. </b> "
---

Put your knowledge into practice with these hands-on exercises. Complete them in order — each builds on the previous one.

---

## Exercise 1: Deploy the Platform Stack

**Objective**: Deploy the full MarTech platform to your AWS account and verify all resources are created.

**Steps**:

1. Clone the repository and install dependencies:

```bash
git clone <repo-url>
cd aws-martech-multiagent
pnpm install
uv sync
```

2. Configure third-party credentials in `packages/infra/config/default.yaml`. At minimum, provide one platform's credentials:

```yaml
deploymentConfig:
  mcp:
    databricks:
      url: "https://your-workspace.cloud.databricks.com"
      token: "dapi-your-personal-access-token"
```

{{% notice info %}}
You only need credentials for the platforms you want to test. Unconfigured platforms are ignored during deployment.
{{% /notice %}}

3. Build all packages:

```bash
pnpm run build:all
```

4. Deploy the stack:

```bash
pnpm exec nx deploy @play-c463-z26-rzy-mar-tech/infra "stack-name/*"
```

5. **Verify** in the AWS Console:
   - CloudFormation → Stacks → Confirm stack status is `CREATE_COMPLETE`
   - Bedrock → AgentCore → Runtimes → Confirm 4 runtimes exist (marketer, databricks, clevertap, talonone)
   - Bedrock → AgentCore → Gateway → Confirm `marketer-gateway` is active with 3 targets
   - Systems Manager → Parameter Store → Confirm parameters exist for each agent

**Checkpoint**: When all 4 runtimes show `Active` status in the AgentCore console, proceed to Exercise 2.

---

## Exercise 2: Chat with the Marketing Agent

**Objective**: Use the Web UI to create a campaign and observe the multi-agent workflow in action.

**Steps**:

1. Find the Web UI URL:
   - CloudFormation → Outputs tab for your stack
   - Look for the `WebUiUrl` output value
   - Open the URL in your browser

2. **Log in** using the admin credentials from your `default.yaml` config

3. Open your browser's **Developer Tools** (F12) → **Network** tab → Filter by `chat`

4. Click **Create Campaign** and give it a name (e.g., "Summer Sale 2026")

5. In the chat panel, type a prompt describing a target audience:

   ```
   Find all premium users in the San Francisco region who have made a purchase in the last 90 days. Estimate the audience size.
   ```

6. **Observe**:
   - The streaming response in the chat UI
   - Network requests showing SSE events (`data: {"type": "text"...}`)
   - `subagent_progress` events when the Databricks Agent is working
   - `tool_use` / `tool_result` expandable panels

7. Respond to the agent's prompts to complete the workflow through Step 2 (and optionally Step 3).

**Checkpoint**: You should see a complete conversation with the agent containing audience details and campaign confirmation.

---

## Exercise 3: Tune an Agent's System Prompt

**Objective**: Modify the Databricks Agent's behavior by editing its system prompt, without redeploying code.

**Steps**:

1. In the Web UI, navigate to the **Configuration** page (`/configuration`)

2. Select the **Databricks Agent** tab

3. View the current system prompt. It instructs the agent to:
   - Discover warehouses, schemas, and tables before querying
   - Poll for long-running results
   - Inform users about S3 download locations

4. **Add a new instruction** at the end of the prompt:

   ```
   Always limit SQL queries to 100 rows maximum unless the user explicitly requests more.
   ```

5. Click **Save**

6. Return to an existing campaign's chat. Send:

   ```
   Show me all users in the database.
   ```

7. **Verify**: The agent should now limit results to 100 rows without you needing to specify this.

8. **Revert**: Change the prompt back to its original content and save.

{{% notice tip %}}
System prompt changes take effect immediately — no redeployment needed. This is the power of dynamic configuration via SSM Parameter Store.
{{% /notice %}}

---

## Exercise 4: Inspect S3 Session Artifacts

**Objective**: Navigate the S3 session bucket to understand the audit trail created by the multi-agent system.

**Steps**:

1. Find the sessions bucket name:
   - CloudFormation → Outputs → Look for `SessionsBucketName`
   - Or search in S3 console for a bucket with "sessions" in the name

2. Navigate into a session folder (e.g., `session-<uuid>/`)

3. **Examine the structure**:

```
session-<uuid>/
├── orchestrator/
│   ├── agent.json
│   └── messages/
│       ├── message_0.json
│       ├── message_1.json
│       └── ...
├── databricks-agent/
│   ├── agent.json
│   └── messages/
└── clevertap-agent/
    ├── agent.json
    └── messages/
```

4. Open `orchestrator/agent.json` and note the agent metadata

5. Open a message file and examine its structure:
   - `role` — user or assistant
   - `content` — the actual message text
   - `createdAt` — timestamp

6. Compare the `orchestrator` and `databricks-agent` folders. Notice how the same session ID connects messages across agents.

**Checkpoint**: You should understand how to trace a complete multi-agent conversation from the S3 artifacts alone.

---

## Exercise 5: Observe MCP Gateway Tool Filtering

**Objective**: Understand how the MCP Gateway filters tools so each agent sees only its own tools.

**Steps**:

1. In the Databricks Agent runtime logs (CloudWatch Logs), search for tool invocation entries

2. Note that tool names are prefixed: `databricks-target___execute_sql`

3. In the Web UI, open the chat and inspect a `tool_use` SSE event. The tool name shown in the UI has the prefix stripped.

4. Check the TalonOne agent logs — verify it never sees `databricks-target___execute_sql`, only `talonone-target___*` tools.

**Expected finding**: The `get_gateway_mcp_client("databricks-target")` factory filters tools by the `databricks-target___` prefix, so each agent only sees its own tools.

---

## Bonus Exercise: Extend the Platform (Conceptual)

**Objective**: Design how you would add a fifth agent (e.g., a Salesforce CRM Agent) to the platform.

**Tasks**:

1. **MCP Server**: Describe the Lambda function you would create. What tools would it expose? (e.g., `list_contacts`, `create_lead`, `get_opportunity`)

2. **Gateway Target**: What would you add to `GatewayConstruct` to register the new target?

3. **Worker Agent**: How would you scaffold a new agent using the shared `create_a2a_app` factory?

4. **Orchestrator tool**: What changes to the Marketing Agent are needed to add the new worker agent tool?

5. **IAM**: What additional permissions are required?

**Discuss your design** with the workshop facilitator or compare with the existing Databricks/CleverTap/TalonOne patterns.

---

## Troubleshooting

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| Agent not responding | Missing Bedrock model access | Verify model access in Bedrock console |
| MCP tool returns 403 | Secrets missing or wrong | Check Secrets Manager for valid credentials |
| Web UI shows blank page | Runtime config not loaded | Run `pnpm exec nx run @project/web-ui:load:runtime-config` |
| A2A call fails | Execution role missing permission | Check IAM role for `bedrock-agentcore:InvokeAgentRuntime` |
| Campaign not appearing | DynamoDB GSI not populated | Check `CampaignActiveIndex` in DynamoDB console |
| CloudFormation stack fails | Service quota exceeded | Request quota increase for Lambda, API Gateway |
