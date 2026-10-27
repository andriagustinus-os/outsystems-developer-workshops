---
title: "JumpStart 4: Embed the Agent in Your App"
description: Integrate your Refund Agent into the web application UI with loading states, API calls, and Markdown rendering.
weight: 50
type: docs
layout: single
created: "2026-08-24"
updated: "2026-08-24"
authors: []
prerequisites: ["jumpstart-3-build-refund-agent"]
duration: "45 minutes"
difficulty: "intermediate"
tags: ["jumpstart", "agentic", "ui", "forge", "integration"]
---

## Overview

Now that your Refund Agent is built and published, you'll embed it directly into your Inventory Management application. The operations team will be able to click a button on any Supplier's detail page and instantly get AI-powered refund analysis.

## What You'll Learn

- How to add an agent as a dependency to your main app
- How to adapt screen layouts for new features
- How to create client-side logic to call an agent
- How to handle loading states in the UI
- How to install and use Forge components (MarkdownRenderer)
- How to display formatted agent output

## Prerequisites

- Complete {{< workshop slug="jumpstart-3-build-refund-agent" label="JumpStart 3: Build a Refund Agent" >}}
- Your Refund Agent must be published

---

## Step 1: Add the Agent as a Dependency

1. Go back to the **Inventory Management** application in ODC Studio
2. Click the **Manage Dependencies** menu
3. Search for **"CallRefundAgent"**
4. Select the **CallRefundAgent** action
5. Click **Add**

![Manage Dependencies menu](image104.png)

![Searching for and selecting CallRefundAgent](image105.png)

---

## Step 2: Adapt the Supplier Detail Screen Layout

1. Open the **"Interface"** tab
2. Under MainFlow, double-click on the **SupplierDetailView** screen (or SupplierDetailsScreen/SupplierView)
3. Click on the container with the Supplier details (click above the SupplierName expression)
4. Select the **Styles** tab
5. Change the **Width** dropdown value to **6 col**

![Opening the SupplierDetailView screen](image106.png)

![Changing container width to 6 col](image109.png)

> ℹ️ **Info**: We're making the existing content narrower to create space for the agent's output on the right side.

---

## Step 3: Add a Loading Button

1. Search for **"Button Loading"** in the widget panel
2. Drag the **Button Loading** widget into the newly created empty area
3. Right-click the Utilities/ButtonLoading label
4. Select **"Enclose in Container"**
5. Select the **Styles** tab of the new container
6. Change **Width** to **5 col** (type "5 col" if it doesn't appear in suggestions)
7. Change **Left Margin** to **1 col**

![Dragging Button Loading widget](image111.png)

![Enclosing in container](image123.png)

![Setting container width and margin](image110.png)

---

## Step 4: Create the IsLoadingAgentResponse Variable

1. In the screen's **Elements** tab
2. Right-click the screen name and select **"Add Local Variable"**
3. Name it: **IsLoadingAgentResponse**
4. Data type should automatically be set as **Boolean**

![Adding IsLoadingAgentResponse local variable](image116.png)

![Variable created with Boolean type](image112.png)

---

## Step 5: Configure the Button and Create the Click Action

1. Click on the **Utilities/ButtonLoading** widget (shown in red)
2. Set the **IsLoading** dropdown to **IsLoadingAgentResponse**
3. Click the text label inside the Button and rename it to **"Refunds"**
4. Click on the **Button** widget
5. Under Properties, open the **OnClick** dropdown
6. Scroll down and select **"New Client Action"**

![Configuring Button Loading and IsLoading property](image113.png)

![Setting button text to Refunds](image115.png)

![Creating new client action for OnClick](image114.png)

![New Client Action created](image122.png)

---

## Step 6: Set Loading State to True

1. Drag an **Assign** widget to the flow
2. Under Properties, set **Variable** = **IsLoadingAgentResponse**
3. Set **Value** = **True**

![Assign widget with IsLoadingAgentResponse = True](image118.png)

![Properties showing the assignment](image133.png)

---

## Step 7: Call the Refund Agent

1. In the top right corner, search for **"CallRefundAgent"**
2. Drag the **CallRefundAgent** action to the flow (after the assign)
3. Set the **UserInput** property to: `GetSupplierById.List.Current.Supplier.SupplierName`
4. Set **SessionId** to: **0**

![Searching for CallRefundAgent](image119.png)

![CallRefundAgent added to the flow](image124.png)

![Setting UserInput and SessionId properties](image128.png)

> ℹ️ **Info**: The CallRefundAgent action is exposed from your Agentic App, allowing you to trigger it from any other agent or OutSystems app.

---

## Step 8: Create the AgentResponse Variable

1. Go back to the **"Interface"** tab
2. Right-click the screen under MainFlow
3. Select **"Add Local Variable"**
4. Name it: **AgentResponse**
5. Data type should be **Text**

![Adding AgentResponse local variable](image116.png)

![AgentResponse variable created](image125.png)

---

## Step 9: Assign the Agent Response

1. Double-click on the **RefundsOnClick** action to open it
2. Drag an **Assign** widget after the CallRefundAgent action
3. Set **Variable** = **AgentResponse**
4. Set **Value** = **CallRefundAgent.Response**

![Assign widget for AgentResponse](image136.png)

![Setting value to CallRefundAgent.Response](image130.png)

![Complete assignment](image129.png)

---

## Step 10: Set Loading State to False

1. Drag another **Assign** widget below the previous one
2. Set **Variable** = **IsLoadingAgentResponse**
3. Set **Value** = **False**

![Final assign setting loading to false](image132.png)

![Complete flow: True → CallAgent → Assign Response → False](image139.png)

> ✅ **Checkpoint**: Your RefundsOnClick flow should be: Assign(Loading=True) → CallRefundAgent → Assign(AgentResponse) → Assign(Loading=False)

---

## Step 11: Install MarkdownRenderer from Forge

1. Go to the **ODC Studio Dashboard**
2. Click the **Install from Forge** button (a web page will open)
3. Search for **"MarkdownRenderer"**
4. Click **Install**
5. Go back to the Inventory Management app

![Forge button in ODC Studio](image134.png)

![Searching for MarkdownRenderer in Forge](image145.png)

![Installing MarkdownRenderer](image159.png)

> ℹ️ **Info**: The OutSystems Forge is a repository of reusable, open-source components and accelerators that bring extra functionality to the platform.

> ⚠️ **Warning**: If the Forge button doesn't work, see the [Forge workaround]({{< relref "/workshops/setting-up" >}}#workaround-forge).

---

## Step 12: Add MarkdownRenderer as a Dependency

1. Open the **Manage Dependencies** menu
2. Search for **"Markdown"**
3. Select the **MarkdownRenderer** block (puzzle icon)
4. Click **Add**

![Adding MarkdownRenderer dependency](image140.png)

---

## Step 13: Add MarkdownRenderer to the Screen

1. Go to the **"Interface"** tab and double-click on **SupplierView** under MainFlow
2. In the top right corner, search for **"Markdown"**
3. Drag the **MarkdownRenderer** block and place it below the button (on the right side)
4. Open the **Markdown** variable dropdown
5. Select **AgentResponse**

![Searching for MarkdownRenderer widget](image135.png)

![MarkdownRenderer placed on screen](image138.png)

![Setting Markdown property to AgentResponse](image137.png)

---

## Step 14: Change Application Timeout

1. On the top left corner, select **Inventory Management** (dropdown → Edit app properties)
2. Select the **Properties** tab
3. Change **Default Timeout** to **60**
4. Click **Close**

![Application properties with timeout](image150.png)

---

## Step 15: Deploy and Test

1. Click **1-Click Publish**
2. Click **"Open in Browser"** to see your app running

![Publishing and opening in browser](image142.png)

![App opened in browser](image147.png)

---

## Step 16: Test Your Agent

1. Navigate to the **"Suppliers"** screen
2. Click on a supplier's name/link to open their detail page
3. Click the **Refunds** button
4. Wait for the agent's output to appear

![Supplier detail page with Refunds button](image146.png)

![Agent output rendered with Markdown](image148.png)

![Formatted refund analysis from the agent](image144.png)

> ⚠️ **Warning**: You might receive different agent output depending on the supplier. Not all suppliers have associated refunds. If a supplier has no refunds, the agent will state: "There are no refunds for the current supplier." Try a different supplier if needed.

---

## Summary

You've successfully embedded your AI agent into the main application! Users can now:

- Open any Supplier's detail page
- Click the "Refunds" button to trigger AI analysis
- See beautifully formatted output powered by the LLM and your live data
- Get instant insights on refund patterns by supplier

## Next Steps

- Continue to {{< workshop slug="jumpstart-5-agent-tools" label="JumpStart 5: Add Tools to Your Agent" >}} (Bonus) to extend your agent with tool-calling capabilities
