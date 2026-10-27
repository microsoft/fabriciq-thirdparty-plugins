# Microsoft Fabric IQ plugins for third-party AI assistants

This repository hosts the **Microsoft Fabric IQ** plugin for third-party AI assistants:
Claude and ChatGPT. Both hosts share one plugin folder and one skill; each host
has its own manifest files.

Microsoft Fabric IQ connects your AI assistant to the Microsoft Fabric IQ remote MCP server,
so you can ask questions about your Power BI reports and semantic models in plain
language and get answers backed by your data. See
[plugins/fabriciq](plugins/fabriciq/README.md) for details.

| Host | Status |
|---|---|
| Claude Code | Available: plugin in this repository |
| Claude (web and desktop) | Available as a custom connector |
| ChatGPT and Codex | In development: package in this repository, not yet published |

## Prerequisites

- A Microsoft Entra work or school account.
- Access to Microsoft Fabric or Power BI, with permission to read the reports and
  semantic models you want to ask about.
- Your organization allows users to consent to the app, or a tenant administrator
  has granted admin consent. See [Admin consent](#admin-consent).

Microsoft Fabric IQ MCP server URL:

```
https://fabriciq.svc.cloud.microsoft/v1/mcp/fabriciq
```

## Configure Claude Code

Microsoft Entra doesn't support dynamic client registration, so Claude Code needs
an Entra app registration with a loopback redirect URI.

1. Register an Entra app as described in step 1 of
   [Configure Claude (web and desktop)](#configure-claude-web-and-desktop), but
   add the redirect URI `http://localhost:8080/callback` (platform
   **Mobile and desktop applications**, or **Web** if you use a client secret).
2. Add the MCP server with the app's client ID. `--scope user` makes it
   available in every project; without it, Claude Code adds the server only to
   the current project. `--client-secret` prompts for the secret and stores it
   in your system keychain or credentials file:

   ```
   claude mcp add --transport http --scope user --client-id <your-app-client-id> --client-secret --callback-port 8080 fabriciq https://fabriciq.svc.cloud.microsoft/v1/mcp/fabriciq
   ```

   Leave out `--client-secret` if the app is a public client.
3. Optionally, install the plugin to add the Microsoft Fabric IQ skill:

   ```
   /plugin marketplace add microsoft/fabriciq-thirdparty-plugins
   /plugin install fabriciq@fabriciq
   ```

4. Restart Claude Code, run `/mcp`, select **fabriciq**, choose
   **Authenticate**, and sign in with your Microsoft Entra account.
5. Run `/mcp` again and confirm that **fabriciq** shows as connected, then ask a
   question, for example "What were total sales by region last quarter?"

## Configure Claude (web and desktop)

Microsoft Entra doesn't support dynamic client registration, so the custom
connector needs an Entra app registration. A tenant administrator does steps 1-2
once; each user then does steps 3-5.

1. In the [Microsoft Entra admin center](https://entra.microsoft.com), register
   an app:
   - **Supported account types:** accounts in any organizational directory
     (multitenant). The Microsoft Fabric IQ server directs sign-in to the
     `organizations` endpoint, which rejects single-tenant apps.
   - **Redirect URI (Web):** `https://claude.ai/api/mcp/auth_callback`
   - **API permissions:** delegated **Power BI Service** permissions
     `Item.Read.All` and `Item.Execute.All`, plus `offline_access`. The
     connection requests the `https://api.fabric.microsoft.com/.default` scope,
     which covers these consented permissions.
   - Create a client secret under **Certificates & secrets**.
2. Grant admin consent if your tenant blocks user consent (see
   [Admin consent](#admin-consent)).
3. In Claude, open **Settings > Connectors** and select **Add custom connector**.
4. Enter the name **Microsoft Fabric IQ** and the MCP server URL. Under
   **Advanced settings**, enter the app's client ID and client secret, then
   select **Add**.
5. Select **Connect**, sign in with your Microsoft Entra account, and enable the
   connector in a conversation from the tools menu.

On Team and Enterprise plans, an organization owner adds the connector under
**Admin settings > Connectors**, and members then connect individually.

## Configure ChatGPT

The ChatGPT plugin uses the portable Agent Plugins format: `plugin.json` and
`mcp.json` at the plugin root, plus the shared `skills/` folder. It isn't in the
ChatGPT plugin directory yet. Until it is, connect Microsoft Fabric IQ as a custom MCP
server or install the plugin locally.

### Connect as a custom MCP server (ChatGPT web)

ChatGPT, like Claude, needs an Entra app because Entra doesn't support dynamic
client registration.

1. Register an Entra app as described in step 1 of
   [Configure Claude (web and desktop)](#configure-claude-web-and-desktop), but
   use the redirect URI that ChatGPT shows when you add the server.
2. In ChatGPT, open **Plugins** (chatgpt.com/plugins), select **+**, then
   **Add custom MCP server**.
3. Enter the name **Microsoft Fabric IQ** and the MCP server URL, choose **OAuth**, and
   enter the app's client ID and client secret.
4. Select **Connect** and sign in with your Microsoft Entra account.
5. In a new conversation, enable **Microsoft Fabric IQ** from the tools menu and ask a
   question.

On Business and Enterprise workspaces, an admin may need to enable custom MCP
servers (developer mode) for the workspace first.

### Install the plugin locally (ChatGPT desktop and Codex)

Use this to test the full plugin, including the skill, before it's published.

1. `plugins/fabriciq/mcp.json` uses the production ChatGPT Entra app client ID.
   To test with your own Entra app, replace it locally and don't commit the
   change.
2. Add your local clone as a marketplace.
   - **Codex CLI:** `codex plugin marketplace add <path-to-your-local-clone>`
   - **ChatGPT desktop:** open the cloned repository as a project. ChatGPT reads
     `.agents/plugins/marketplace.json` from the repository root.
3. Install **Microsoft Fabric IQ** from the marketplace. When prompted, enter the app's
   client secret (it's sent with `client_secret_post` and is never stored in
   this repository), then sign in with your Microsoft Entra account. If your
   client doesn't prompt for a secret, use
   [Connect as a custom MCP server](#connect-as-a-custom-mcp-server-chatgpt-web)
   instead.
4. Ask a question, for example "What were total sales by region last quarter?"

### Build the submission package

The OpenAI plugin directory takes a ZIP with `plugin.json` at its root. Build it
with:

```powershell
./scripts/build-chatgpt-zip.ps1
```

The script writes `dist/fabriciq-chatgpt.zip` with only the files ChatGPT needs
(`plugin.json`, `mcp.json`, `skills/`, `assets/`, `README.md`, `LICENSE`). It
fails while `mcp.json` has a missing or TODO client ID, or `plugin.json` still
has the `REVIEW_*` placeholders; pass `-AllowPlaceholders` for a local test
build.

`mcp.json` uses the OpenAI `extensions["com.openai"]` block for OAuth, so it
doesn't validate against the strict agent-plugins.org schema. That's expected.

For ChatGPT, `mcp.json` points at `https://api.fabric.microsoft.com/v1/mcp/fabriciq`
and sets `resource` to `https://api.fabric.microsoft.com` with the
`https://api.fabric.microsoft.com/.default` scope. The Fabric IQ host advertises
a path-based resource (`.../v1/mcp/fabriciq`) that Entra rejects at token
exchange (AADSTS9010010), and the portal doesn't apply the `resource` override
there. Switch back once the Fabric IQ host advertises
`https://fabriciq.svc.cloud.microsoft` as its resource.

`baseScopes` is only `offline_access`. Entra rejects a request with two
`.default` scopes (AADSTS70011).

Before submitting at platform.openai.com/plugins:

- The client secret is entered in the OpenAI portal, never in this repository.
- Replace `assets/icon.png` and `assets/logo.png` with the final Microsoft Fabric IQ
  artwork, and prepare screenshots.
- The `review` test cases in `plugin.json` use the "2020 product inventory
  report" and "product model - terechen" semantic model in the reviewer test
  workspace. Update them if that workspace changes.
- Provide a demo recording URL (`demo_recording_url`) of the plugin working;
  it's required for MCP review.
- Provide a dedicated reviewer test account that signs in without MFA and has
  access to the test workspace.
- Confirm the terms of service URL in `.claude-plugin/plugin.json`.
- Verify the MCP server domain at `/.well-known/openai-apps-challenge`.

For the Claude directory, submit the plugin and also submit the MCP server as a
connector.

## Admin consent

If your tenant blocks user consent, sign-in shows **Admin approval required**. A
tenant administrator can grant consent for the whole tenant by opening the
following URL, replacing the placeholders, and then accepting:

```
https://login.microsoftonline.com/<tenant-id>/adminconsent?client_id=<app-client-id>
```

## Troubleshooting

| Symptom | What to do |
|---|---|
| **Admin approval required** at sign-in | Ask a tenant administrator to grant admin consent. |
| The connector stops working after a while | Reconnect it in Claude (**Settings > Connectors**) or ChatGPT (**Plugins**), or re-run **Authenticate** from `/mcp` in Claude Code. |
| A report or semantic model isn't found | Confirm you have at least read access to it in Microsoft Fabric. Microsoft Fabric IQ only sees items you can already access. |
| Answers omit some rows or columns | Row-level and object-level security on the semantic model still apply. |

## Repository layout

```
.claude-plugin/marketplace.json   Claude marketplace manifest
.agents/plugins/marketplace.json  ChatGPT and Codex marketplace manifest (local testing)
plugins/fabriciq/                 The Microsoft Fabric IQ plugin
  .claude-plugin/plugin.json      Claude plugin manifest
  .mcp.json                       Claude MCP server configuration
  plugin.json                     ChatGPT plugin manifest (Agent Plugins format)
  mcp.json                        ChatGPT MCP server and OAuth configuration
  assets/                         Icon and logo, shared by both hosts
  skills/fabriciq/                Microsoft Fabric IQ skill, shared by both hosts
scripts/build-chatgpt-zip.ps1     Builds the ChatGPT submission ZIP
```

Keep the skill host-neutral: it ships to both Claude and ChatGPT. When you change
the version, update it in both `plugin.json` files and in
`.claude-plugin/marketplace.json`.

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Attribution

The bundled skill is adapted from
[microsoft/skills-for-fabric](https://github.com/microsoft/skills-for-fabric) (MIT).

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
