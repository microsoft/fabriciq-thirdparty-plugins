# Power BI

Work with Power BI semantic models from a conversation with Claude.

## What it does

The plugin adds the **semantic-model-authoring** skill and a connection to the Power BI MCP server. The skill guides Claude through inspecting semantic models, explaining their tables, relationships and measures, writing and testing DAX queries, and preparing, reviewing and testing model changes. Report authoring isn't included yet.

## Use it

1. Install the plugin, then open its **Connectors** tab and connect **powerbi**. Sign in with a Microsoft work account that has a Power BI license, in an organization that allows the Power BI Model Context Protocol server.
2. Ask Claude about a model, for example: "Explain the measures in the Sales model in my Finance workspace" or "Check this DAX query against the Sales model".

Claude changes a model only when you ask it to. What it can do depends on your Power BI permissions. Model work needs a model connection that lasts from one tool call to the next; Claude can start a new session for each call, and when the connection is lost, Claude tells you instead of retrying.

## Data

The plugin itself stores nothing and contains no credentials. When you use it, Claude sends your requests and the model details it works with to the Power BI MCP server at api.fabric.microsoft.com, signed in as you. The server returns model metadata and query results to the conversation. See the [Microsoft privacy statement](https://go.microsoft.com/fwlink/?linkid=2282651).
