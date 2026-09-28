---
title: "Building API-based AI Agent and integration with A2A"
description: Build an agent with Agent Workbench, connect it to an external agent over A2A, and expose the whole thing as a REST API.
weight: 20
type: docs
layout: single
created: "2026-09-05"
updated: "2026-09-06"
authors: ["donnie"]
prerequisites: ["setting-up"]
duration: "65 minutes"
difficulty: "intermediate"
tags: ["agentic", "ai", "integration", "a2a", "rest-api"]
---

## Overview

<a href="https://www.outsystems.com/low-code-platform/agentic-ai-workbench" target="_blank">Agent Workbench</a> is the feature in OutSystems for designing and building agentic AI systems in ODC Studio. In this workshop, you'll build an agentic app with it, connect that agent to a completely separate, externally-hosted agent using the **Agent-to-Agent (A2A) protocol**, and finish by exposing your OutSystems agent as a REST API so any external system can call it directly.

By the end, you'll have a single HTTP endpoint that, behind the scenes, triggers your OutSystems agent, which in turn calls out to a real external AI agent running on entirely different infrastructure — and returns one combined response.

## What You'll Learn

- How to create an agentic app with Agent Workbench
- How to recognize and diagnose an agent that can't reach an external system yet
- How to add an A2A connection to an external agent and review its agent card
- Why handing an AI model a complex, structured tool input is unreliable, and how to wrap it behind one simple server action
- How to expose an agent as a REST API with built-in authentication
- How to test a live agent-to-agent integration end-to-end with Postman

## Prerequisites

- Complete {{< workshop slug="setting-up" label="Setting Up for OutSystems Workshops" >}}
- <a href="https://www.postman.com/downloads/" target="_blank">Postman</a> (or any REST client) installed
- The API key for this workshop's external agent — ask your facilitator. You'll enter it directly into the A2A connection's authentication settings in the ODC Portal (Step 4) — it doesn't belong anywhere else, including app logic, screenshots, or anything you commit

---

## Context

A support operations team at a fictional retailer, Meridian Retail, wants any internal tool — the helpdesk, an internal dashboard, a Slack bot — to be able to send a customer complaint to one API and get back a drafted, empathetic response. Someone else's team has already built and hosted an AI agent that's good at exactly this: drafting support responses from a customer's history and flagging anything too sensitive for a human to review.

Your job isn't to rebuild that — it's to build the OutSystems-side agent that owns the request, delegates the hard part to the specialist agent, and exposes the result as a clean, authenticated API.

---

## Step 1: Create an Agentic App with Agent Workbench

1. In <a href="https://www.outsystems.com/low-code-platform/developer-cloud" target="_blank">ODC Studio</a>, click **Create** and select **Agentic app**

![ODC Studio create menu with Agentic app selected](./images/building-api-based-ai-agent-and-integration-with-a2a-1.png)

2. Name the app **"Support Response Router"**

![Naming the agentic app Support Response Router](./images/building-api-based-ai-agent-and-integration-with-a2a-2.png)

3. (Optional) Add a description, e.g.: *"Routes customer complaints to a specialized external agent for response drafting, and returns the result."*
4. Choose "TrialClaudeHaiku4_5" for the AI model and select on "Create agentic app"

![Choosing the AI model and creating the agentic app](./images/building-api-based-ai-agent-and-integration-with-a2a-3.png)

> ℹ️ **Info**: This is the same underlying feature — Agent Workbench — used in {{< workshop slug="jumpstart-3-build-refund-agent" label="JumpStart 3" >}} to build the Refund Agent. This workshop's agent is a separate, unrelated app — you don't need to have completed that one.

---

## Step 2: Define the OutSystems-Side Agent's Job

1. In the **Logic** tab, open **AgentFlow** and double-click the **BuildMessages** action

![Opening BuildMessages inside AgentFlow](./images/building-api-based-ai-agent-and-integration-with-a2a-4.png)

2. Click on the **SystemMessage** assign, then find **SystemMessageContent.ContentText** and double-click the **`x.y`** expression icon below it (the small value-edit symbol that opens the expression editor)

![Opening the expression editor for SystemMessageContent.ContentText](./images/building-api-based-ai-agent-and-integration-with-a2a-5.png)

3. Replace the default text with instructions that keep this agent's scope narrow — routing and returning, not drafting:

```text
"You are a support routing agent.
Your only job is to call the SendMessageWrapper tool with the customer complaint text, then output the result.
When you call SendMessageWrapper, the tool will return a response containing artifacts with parts.
One part has kind text and contains the drafted customer response.
Your reply must be ONLY the exact text from that text part. Copy it word for word.
Do not add any introduction, explanation, or commentary before or after it.
Do not say the tool was called.
Do not summarize.
Output nothing except that exact drafted response text."
```

> 💡 **Tip**: The instructions are thin on purpose — all the actual drafting intelligence lives in the external agent; this agent's job is routing, not reasoning. Note the instructions already name `SendMessageWrapper`, which you haven't built yet. By design: Steps 5–7 build exactly what this prompt describes.

---

## Step 3: Publish and Test the Baseline Agent

Before touching A2A at all, publish and test what you have. It will fail in a specific, informative way — and that's exactly what you're looking for.

1. Click **1-Click Publish** to publish your agentic app
2. Create a test agent app

![Creating a test agent app](./images/building-api-based-ai-agent-and-integration-with-a2a-6.png)

![Creating a test agent app](./images/building-api-based-ai-agent-and-integration-with-a2a-7.png)

![Creating a test agent app](./images/building-api-based-ai-agent-and-integration-with-a2a-8.png)

3. Open the agent's built-in test/preview panel
4. Send this complaint:

```text
I have a complaint from customer ID CUST-001. They ordered a wireless headset last week,
and after 3 days of use, the left earcup stopped working. They're frustrated because
they need it for work calls.
```

![Complaint typed into the test panel, ready to send](./images/building-api-based-ai-agent-and-integration-with-a2a-10-1.png)

5. The agent responds with something like: *"I don't have access to an external Customer Support Responder agent to call."*

![Agent's response highlighting that it doesn't have access to call the external agent](./images/building-api-based-ai-agent-and-integration-with-a2a-10.png)

> ℹ️ **Info — what went wrong**: The agent's instructions (Step 2) tell it to call `SendMessageWrapper`, but no A2A connection exists yet, and neither the connection's action nor the wrapper have been built. The agent isn't broken; it's correctly telling you there's nothing to call yet. Fixing this takes several steps, in order: create the A2A connection (Step 4), add its `SendMessage` action to the app (Step 5), understand why wiring that action in directly is fragile (Step 6), build a small wrapper action that avoids that fragility (Step 7), and wire the wrapper in (Step 8).

> ✅ **Checkpoint**: A coherent response acknowledging it can't make external calls means your agent itself is working correctly. The steps that follow are what give it something to call and tell it to call it — properly.

---

## Step 4: Add an A2A Connection to the External Agent

This creates the connection you diagnosed as missing in Step 3 — but on its own, it only gets you partway there.

1. In the **ODC Portal**, go to **Connections** then click **Create connection**

![Connections page with Create connection button](./images/building-api-based-ai-agent-and-integration-with-a2a-11.png)

2. Select **A2A** then click **Confirm**

![Selecting A2A as the connection provider](./images/building-api-based-ai-agent-and-integration-with-a2a-12.png)

3. Name the connection **"CustomerSupportResponder"**. Then, enter the base URL of the external agent's A2A endpoint: `https://c4zehdwqhc.execute-api.ap-southeast-1.amazonaws.com/prod/` and click **Get details** to fetch the agent card

![Naming the connection and fetching the agent card](./images/building-api-based-ai-agent-and-integration-with-a2a-13.png)

4. Open the **Skills** tab and review what the agent card declares — you should see a `draft-support-response` skill. Then, select **API Key** as the authentication mechanism. You need to enter the key value your facilitator gave you. Then, click **Test connection** to confirm everything is configured correctly and click on **Save**.

![Skills tab, API Key authentication, and successful test connection](./images/building-api-based-ai-agent-and-integration-with-a2a-14.png)

> ℹ️ **Info**: The agent card is a JSON document the external agent publishes at a well-known path (`/.well-known/agent-card.json`) describing its identity, skills, and how to authenticate to it. ODC's A2A connector reads this directly — you never have to hand-write an integration against a spec you can't see.

> ℹ️ **Info**: Note the pre-filled **Key location** and **Key name** — these come directly from the agent card's declared security scheme. This particular agent expects its key in a header named `x-api-key`; a different agent's card could declare a query parameter or a different header name entirely, and ODC would fill in different values here.

> ⚠️ **Warning**: Never commit the API key to source control or paste it into shared documents. Treat it the same as any other production secret.

> ⚠️ **This isn't the full fix yet**: Creating the connection doesn't wire it into your agent's logic. The next four steps do that: add `SendMessage` as a public element (Step 5), understand why wiring it directly is fragile (Step 6), build a wrapper (Step 7), and wire the wrapper in (Step 8).

---

## Step 5: Add the A2A Connection's `SendMessage` Action to Your App

Before `SendMessage` can be used anywhere in your agent's logic — including inside the wrapper you'll build shortly — it has to be added to this app as a public element. This is a one-time reference step, separate from actually calling it.

1. In ODC Studio, click **Add public element**, then click **All sources**

![All sources option when adding a public element](./images/building-api-based-ai-agent-and-integration-with-a2a-15.png)

2. Type **CustomerSupportResponder** to filter down to the connection you created in Step 4

![Filtering All sources for the CustomerSupportResponder connection](./images/building-api-based-ai-agent-and-integration-with-a2a-16.png)

3. Select **SendMessage** and click **Add**

![Selecting the SendMessage action to add](./images/building-api-based-ai-agent-and-integration-with-a2a-17.png)

4. Confirm the A2A integration now appears in your agentic app's Logic tab

![SendMessage listed as a public element in the Logic tab](./images/building-api-based-ai-agent-and-integration-with-a2a-18.png)

> ℹ️ **Info**: Adding `SendMessage` here makes it referenceable elsewhere in the app — it doesn't call it or wire it into anything yet. Next, before using it, it's worth understanding *why* you shouldn't just wire it directly into Action Calling as-is.

> ✅ **Checkpoint**: `SendMessage` should now be visible as a dependency under your agentic app's Logic tab — that's what makes it draggable into the wrapper you'll build in Step 7.

---

## Step 6: Understand Why Wiring `SendMessage` Directly Fails

Before building anything, it's worth understanding a real trap here — because the naive approach looks like it should work, and only fails intermittently.

The connection's `SendMessage` action doesn't take a simple message parameter. Its `UserMessage` input is a **List of Records**, and each record represents one A2A "Part." Per the action's own description: *"A UserMessage can only include one content type (TextContent, DataContent, or FileContent)."*

![UserMessage record structure showing TextContent, DataContent, and FileContent as mutually exclusive fields](./images/building-api-based-ai-agent-and-integration-with-a2a-19.png)

To send both a complaint and a customer identifier — which the external agent requires — you need a **two-item list**: one record with `TextContent` set to the complaint, and a separate record with `DataContent` set to a JSON string like `{"customer_id":"CUST-001"}`.

If you add `SendMessage` directly as a tool and leave `UserMessage` entirely "Generated by AI," the LLM has to construct that exact nested, two-item shape from scratch on every single call. A better prompt won't fix this reliably — the model simply can't be trusted to produce that exact nested shape on every call.

> ⚠️ **Warning**: This is what it looks like when it goes wrong. Sometimes the model gets the text Part right but drops the data Part (or vice versa) — there's no single, consistent failure. Check **ODC Portal → Monitoring → Traces** and you'll find it logged like this:

```text
Error
Support Response Router
Revision 3
OS-ABRS-CONN-40013 - There was an exception thrown in server action 'SendMessage'
while executing a tool call: 'Missing text content in TextPart'

Error
Support Response Router
Revision 3
OS-ABRS-CONN-40013 - Missing text content in TextPart
```

> ℹ️ **Info**: The trace shows this twice — once as the outer tool-call exception, once as the inner root-cause message. Both point at the same thing. Depending on where you're looking, you may also see this surfaced as a different code in the live test panel — same underlying failure either way.

The external agent is doing exactly the right thing — rejecting a malformed request and saying precisely what's missing. The problem isn't the agent or the prompt; it's structural: don't expect the model to hand-build a nested record list on every call.

> ✅ The fix: never let the LLM build that nested structure. Wrap it behind a server action with flat parameters, and build the structure yourself with explicit, deterministic logic. That's Step 7.

---

## Step 7: Build `SendMessageWrapper`

This server action hides the two-Part structure behind two simple, flat inputs — `TextContent` and `CustomerId` — that an LLM can populate reliably.

1. In the **Logic** tab, right-click **Server Actions** → **Add Server Action** → name it **`SendMessageWrapper`**

![Add Server Action dialog naming SendMessageWrapper](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-1.png)

2. Add input parameters:
   - **`TextContent`** (Text, mandatory) — description: *"The customer's complaint or feedback text — always populate this with the customer's original message, never leave it empty."*
   - **`CustomerId`** (Text, mandatory)
   - **`TaskId`** (Text, optional)
   - **`ContextId`** (Text, optional)
3. Add output parameters (all Text):
   - **`ResponseContent`**
   - **`TaskStatus`**
   - **`OutTaskId`**
   - **`OutContextId`**
4. Add three local variables:
   - Two of type **`UserMessage`** (the same Record type from Step 6) — name them `UserMessageRecord` and `UserMessageRecordData`
   - One of type **`UserMessage List`** — name it `UserMessageList`

![Input, output parameters, and local variables configured on SendMessageWrapper](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-2.png)

5. Drag an **Assign** node: `UserMessageRecord.TextContent = TextContent` — this is your text Part

![Assign node setting UserMessageRecord.TextContent](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-3.png)

6. Drag another **Assign** node to hand-build the JSON for the data Part:

```text
UserMessageRecordData.DataContent = "{" + Chr(34) + "customer_id" + Chr(34) + ":" + Chr(34) + CustomerId + Chr(34) + "}"
```

![Dragging the Assign node for UserMessageRecordData.DataContent](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-4.png)

![Expression editor showing the Chr(34) JSON-building expression for UserMessageRecordData.DataContent](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-5.png)

> ℹ️ **Info**: `Chr(34)` returns a literal double-quote character. There's no native JSON-serialization shortcut for a single key-value pair here, so this expression builds `{"customer_id":"CUST-001"}` character by character. It looks fussy, but it's fully deterministic — no AI-inference involved.

7. Drag a **Run Server Action** node onto the flow, and in the **Select Action** dialog search for and select the built-in **ListAppend**:

![Select Action dialog searching for and selecting ListAppend](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-5-1.png)

8. Name this ListAppend node **`AppendUserMessage`**. For **List** use `UserMessageList` and for **Element** use `UserMessageRecord` — this appends your text Part:

![ListAppend configured as AppendUserMessage with List and Element parameters set](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-6.png)

9. Drag **Run Server Action** again and add another **ListAppend** to append `UserMessageRecordData` onto the *same* `UserMessageList` — this is your data Part

![Second ListAppend appending UserMessageRecordData to UserMessageList](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-7.png)

10. Drag the real **`SendMessage`** action (added as a public element in Step 5) onto the flow, mapping `TaskId`, `ContextId`, and `UserMessage = UserMessageList`

![SendMessage action added to the flow with TaskId, ContextId, and UserMessage mapped](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-8.png)

11. Add final **Assign** nodes: `ResponseContent = SendMessage.Content`, `TaskStatus = SendMessage.Status`, `OutTaskId = SendMessage.Id`, `OutContextId = SendMessage.Context.ContextId`

![Final Assign nodes mapping SendMessage outputs to ResponseContent, TaskStatus, OutTaskId, OutContextId](./images/Step7/building-api-based-ai-agent-and-integration-with-a2a-9.png)

> ✅ **Checkpoint**: Your `UserMessageList` now always contains exactly two records — one text, one data — regardless of what the LLM does. The only things left to AI-inference are two flat Text values.

---

## Step 8: Wire `SendMessageWrapper` into Action Calling

`SendMessageWrapper` is a local server action in this same app, so — unlike the connection's `SendMessage` action — it doesn't need to be added as a public element first. It's already available to Action Calling.

1. Open **AgentFlow** and double-click **CallTrialClaudeHaiku4_5**

![Opening CallTrialClaudeHaiku4_5 inside AgentFlow](./images/Step8/building-api-based-ai-agent-and-integration-with-a2a-1.png)

2. On the **Action calling** tab, click **Add action** and select **SendMessageWrapper**

![Add action button on the Action calling tab](./images/Step8/building-api-based-ai-agent-and-integration-with-a2a-2.png)

![SendMessageWrapper selected as the action](./images/Step8/building-api-based-ai-agent-and-integration-with-a2a-3.png)

3. Leave `TextContent` and `CustomerId` as **Generated by AI** — they're simple flat values now, which is exactly what AI-inference handles reliably (unlike the raw `UserMessage` list from Step 6), then publish your agent

![SendMessageWrapper parameters — ContextId, CustomerId, TaskId, TextContent — all left as Generated by AI](./images/Step8/building-api-based-ai-agent-and-integration-with-a2a-5.png)

> 💡 **Tip**: The LLM decides when to invoke this action based on your agent's instructions from Step 2 — you're not hand-coding an if/else branch, you're describing the job and letting the agent orchestrate the call.

---

## Step 9: Re-Test the Integration Inside ODC Studio

1. Open the agent's test/preview panel
2. Send the **exact same complaint** from Step 3:

```text
I have a complaint from customer ID CUST-001. They ordered a wireless headset last week,
and after 3 days of use, the left earcup stopped working. They're frustrated because
they need it for work calls.
```

3. Confirm the response now includes a real drafted response, a human-review flag, and an escalation reason when applicable — not the "I can't make external calls" message from Step 3

![Test panel showing a real, personalized drafted response after wiring the A2A call](./images/Step9/building-api-based-ai-agent-and-integration-with-a2a-1.png)

> ✅ **Checkpoint**: A specific, personalized response — referencing customer's actual order history, not a generic fallback — confirms the fix worked end to end: the connection is live, the wrapper reliably built both Parts, and the agent is calling it correctly.

---

## Step 10: Expose the Agent as a REST API

1. In the **Logic** tab, open **Integrations → REST**. Then, right-click and select **Expose REST API**

![REST menu with Expose REST API option selected](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-1.png)

2. Name it **`SupportResponseAPI`**. Ensure the **Advanced > Documentation** is set to **Yes**.

![Naming the REST API and setting Documentation to Yes](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-5.png)

3. Right-click **`SupportResponseAPI`** and select **Add REST API Method**

![Adding a REST API method to the SupportResponseAPI](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-2.png)

4. Name the method **`Complaints`**, set its **URL Path** to `/complaints`, and set **HTTP Method** to **POST**

![Complaints method created with URL Path /complaints and HTTP Method set to POST](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-3.png)

5. **Expose REST API** creates an empty method — it does not inherit `SendMessageWrapper`'s parameters automatically. Give it a proper structured request and response instead of a placeholder input:

   - Switch to the **Data** tab (the cylinder/database icon in the left-side toolbar). Find the **Structures** folder, right-click it, and select **Add Structure**

   ![Right-click menu on the Structures folder with Add Structure selected](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-6.png)

   - Name it `ComplaintsRequest`, then right-click the new structure and select **Add Structure Attribute**

   ![ComplaintsRequest structure created, with the Add Structure Attribute context menu open](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-7.png)

   - Add each attribute, setting its **Name** and **Data Type**, and marking it **Mandatory** where it isn't optional: `CustomerId` (Text, mandatory), `TextContent` (Text, mandatory), `ContextId` (Text, optional), `TaskId` (Text, optional)

   ![CustomerId attribute configured with Data Type Text and Is Mandatory set to Yes](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-8.png)

   - Repeat the same process for a second structure, `ComplaintsResponse`, with attributes `ResponseContent` (Text), `ContextId` (Text), `TaskId` (Text), `TaskStatus` (Text)

   ![Both ComplaintsRequest and ComplaintsResponse structures fully built with all their attributes](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-9.png)

   - Back in the **Logic** tab, open the `Complaints` method and right-click it to **Add Input Parameter**. Name it **`Request`** and set its **Data Type** to `ComplaintsRequest` — this becomes the JSON request body

   ![Right-click menu on the Complaints method with Add Input Parameter and Add Output Parameter](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-10.png)

   - Right-click again and select **Add Output Parameter**. Name it **`Response`**

   ![Right-click menu with Add Output Parameter selected, after the Request input parameter has already been added](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-11.png)

   - Set `Response`'s **Data Type** to `ComplaintsResponse`, and confirm **Send In** is set to **Body** — this becomes the JSON response body

   ![Response output parameter configured with Data Type ComplaintsResponse and Send In set to Body](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-12.png)

> ℹ️ **Info**: A single structure-typed parameter, rather than several flat parameters, is what lets ODC serialize the whole request and response as one JSON object matching the shapes above, instead of flattening every field onto the top level.

6. Between the Start and End nodes, add a **Run Server Action** node

![Adding a node between Start and End, with Server action as one of the options](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-13.png)

7. Search for and select **`SendMessageWrapper`**

![Select Action dialog with SendMessageWrapper highlighted](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-14.png)

8. Map each of `SendMessageWrapper`'s inputs from the `Request` parameter:
   - `CustomerId` ← `Request.CustomerId`
   - `TextContent` ← `Request.TextContent`
   - `ContextId` ← `Request.ContextId`
   - `TaskId` ← `Request.TaskId`

![SendMessageWrapper node with its inputs mapped from the matching Request fields](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-15.png)

9. Add an **Assign** node before the End node to populate the output:
   - `Response.ResponseContent` ← `SendMessageWrapper.ResponseContent`
   - `Response.ContextId` ← `SendMessageWrapper.OutContextId`
   - `Response.TaskId` ← `SendMessageWrapper.OutTaskId`
   - `Response.TaskStatus` ← `SendMessageWrapper.TaskStatus`

![Assign node mapping each Response field from the corresponding SendMessageWrapper output](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-16.png)

10. You have an option to make your API more secure by adding authentication. In the REST service properties, set **Authentication Type** to **Basic Authentication**. This topic will be covered in another workshop.

![REST API Authentication dropdown open, choosing between None, Basic, and Custom](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-4.png)

11. Publish the app


12. You can view the auto-generated documentation for your method at:

```text
https://YOUR_TENANT_URL/SupportResponseRouter/rest/SupportResponseAPI/#/SupportResponseAPI/Complaints
```

![REST API documentation page for SupportResponseAPI, showing the Complaints POST method with example request and response JSON](./images/Step10/building-api-based-ai-agent-and-integration-with-a2a-17.png)


> 💡 **Tip**: For production, ODC's documented pattern is JWT-based token authentication with a dedicated token-issuing service — more setup, but it separates "who issued this caller's identity" from "does this API trust it." Worth knowing exists; out of scope here.


---

## Step 11: Test the Exposed API Externally with Postman

1. In Postman, create a new **POST** request to your published API's URL
2. In the request body, send the same complaint again — `ContextId` and `TaskId` are optional, so they can be omitted:

```json
{
  "TextContent": "I have a complaint from customer ID CUST-001. They ordered a wireless headset last week, and after 3 days of use, the left earcup stopped working. They're frustrated because they need it for work calls.",
  "CustomerId": "CUST-001"
}
```

3. Send the request

![Postman POST request to /complaints with a JSON body, and a 200 OK response containing the drafted reply](./images/Step11/building-api-based-ai-agent-and-integration-with-a2a-1.png)

> ✅ **Checkpoint**: A 200 response containing the drafted reply confirms the full chain works: Postman → your exposed REST API → your OutSystems agent → `SendMessageWrapper` → A2A → the external agent → back out again.

---

## 🥳 Congratulations!

You've built an OutSystems agent that doesn't try to do everything itself — it routes a request to a specialist agent hosted on completely different infrastructure, then hands the combined result back out through a REST API secured with real authentication. Specifically, you:

- Created an agentic app with Agent Workbench
- Deliberately tested it before any A2A setup existed, and learned to read its failure as a diagnosis, not a bug
- Connected it to an external agent via A2A, using its published agent card, and added its `SendMessage` action to the app
- Understood *why* wiring a complex, structured tool input directly to an LLM is unreliable — not just that it failed, but the actual mechanism
- Built a wrapper server action that hides that complexity behind two simple, flat parameters
- Exposed the result as a Basic-Authenticated REST API
- Verified the entire chain end-to-end with Postman

## Next Steps

- If you haven't already, {{< workshop slug="jumpstart-3-build-refund-agent" label="JumpStart 3: Build a Refund Agent" >}} covers Agent Workbench from the ground up with a locally-grounded agent — useful context, though not required for what you just built.
