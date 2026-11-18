# Microsoft Fabric IQ

Access your Power BI reports and semantic models in Microsoft Fabric and get answers grounded in your governed data.  Requires a Microsoft Power BI or Microsoft Fabric license.

Microsoft Fabric IQ finds the reports and semantic models you can access, understands their
tables, measures and relationships, writes and runs the DAX query for you, and
explains the result. You don't need to know DAX or where the data lives.

## What you can ask

- "What were total sales by region last quarter?"
- "Which products had the biggest drop in margin compared to last year?"
- "Show the top 10 customers by revenue in the Contoso Sales report."
- "How many open support tickets do we have per priority?"

If more than one report or model matches your question, Microsoft Fabric IQ asks you which
one you mean.

## Requirements

- A Microsoft Entra work or school account with a Microsoft Power BI or Microsoft Fabric license.
- Access to Microsoft Fabric or Power BI, with permission to read the reports
  and semantic models you want to ask about.

## Getting started

1. Register a Microsoft Entra app. Entra doesn't support dynamic client
   registration, so every host needs its own app registration and client ID.
   See the setup steps for your host in the
   [repository README](https://github.com/microsoft/fabriciq-thirdparty-plugins#readme).
   In Claude Code, add the Microsoft Fabric IQ MCP server with your app's client ID and
   `--scope user`.
2. Install the Microsoft Fabric IQ plugin.
3. Connect the Microsoft Fabric IQ connector when prompted, and sign in with your
   Microsoft Entra account.
4. Ask a question about your data.

## Data and privacy

Microsoft Fabric IQ runs with your own delegated permissions. It only sees reports and
semantic models you already have access to, and existing row-level and
object-level security still applies. It reads data and runs queries; it does not
create, change or delete reports, models or measures.

Your questions and the query results are sent between your AI assistant (Claude
or ChatGPT) and Microsoft Fabric. The plugin itself stores no data and contains
no credentials. See the
[Microsoft Privacy Statement](https://go.microsoft.com/fwlink/?LinkId=521839).

## Support

- Documentation: [Microsoft Fabric documentation](https://learn.microsoft.com/fabric/)
- Help: [Microsoft Fabric support](https://support.fabric.microsoft.com/support)

## License

MIT; see [LICENSE](LICENSE). The bundled skill is adapted from
[microsoft/skills-for-fabric](https://github.com/microsoft/skills-for-fabric)
(MIT, Microsoft Corporation) at commit bc42aaf734771bf752c1d7082dd91e5a6ca72b9f.
