# Branch AI & MCP Gateway

Welcome to Branch's centralized hub for public AI resources and integrations. This repository bridges the gap between AI assistants and the Branch platform by exposing two distinct integration vectors:

1. **Automated MCP Infrastructure:** A live, cloud-hosted Model Context Protocol (MCP) server for real-time API reference discovery.
2. **Contextual Claude Skills:** Specialized, pre-configured instruction sets that guide AI models through compliant engineering and onboarding patterns.

---

## 1. Automated Model Context Protocol (MCP) Server

Branch runs a managed cloud-native MCP server directly through our core documentation platform. This server securely exposes read-only tools allowing AI agents to dynamically query, search, and parse the live Branch API reference. 

Because this is hosted via the cloud, there is no local environment configuration required—no local runtime dependencies, Node modules, scripts, or manual compilation.

* **Endpoint Gateway URL:** https://docs.branchapp.com/mcp
* **Transport Mechanism:** Streamable HTTP

### Integration & Configuration

To expose Branch's documentation tools to an MCP-capable client (such as Claude Desktop), add the endpoint to your ecosystem configuration file. 

For **Claude Desktop**, append the following block to your configuration schema (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS, or `%APPDATA%\Claude\claude_desktop_config.json` on Windows) and restart the application:

```json
{
  "mcpServers": {
    "branch-docs": {
      "type": "http",
      "url": "https://docs.branchapp.com/mcp"
    }
  }
}
```

Once initialized, your AI assistant will programmatically discover, surface, and execute the server's documentation lookups autonomously during active conversations.

---

## 2. Enterprise Claude Skills (Manual Provisioning)

The `/skills` directory contains packaged Claude Skills. These are explicit, highly structured behavioral guardrails that train an AI model to architect Branch integrations safely and correctly (e.g., properly routing sensitive KYC data outside of partner applications, utilizing standard webhooks, and optimizing global payout architectures).

> ### Important Deployment Note
> Unlike the MCP Server, Claude Skills **do not self-load**. Due to the sandboxed, contextual design of LLM workspaces, an AI assistant cannot discover, activate, or persist these instructions simply by reading or crawling this repository. Connecting the MCP server does not import these skills; they operate on a separate infrastructure layer. They must be manually uploaded into your Claude workspace environment using the steps below.

### Workspace Deployment (Individual Accounts)

1. Navigate to the `/dist` directory in this repository and download the compiled `.zip` archive for your desired skill.
2. In your Claude interface, open **Customize ➔ Skills**.
3. Upload the downloaded `.zip` file to activate it within your account.
*Note: Ensure "Code Execution" is toggled on within your individual Claude workspace settings.*

### Enterprise Distribution (Organization-Wide)

To provision a skill globally across an entire enterprise workspace so individual users do not have to configure it manually:
1. An **Organization Owner** must navigate to **Organization Settings ➔ Skills**.
2. Upload the target `.zip` archive from the `/dist` folder.
3. The skill will automatically cascade down and activate for all registered team accounts under the corporate domain.

---

## Repository Structure: `/skills` vs `/dist`

To maintain open-source transparency while providing seamless deployment, this repository separates source materials from production assets:

* **`/skills` (Source Framework):** Contains editable, human-readable markdown manifests (`SKILL.md`) and contextual reference schemas. This is the development workspace where updates, code contributions, and peer reviews occur.
* **`/dist` (Production Builds):** Contains the compiled, ready-to-upload `.zip` deployment packages. Each archive bundles the corresponding markdown source and linked assets into a single-step upload package compatible with Claude's ingestion engine.

*When source files in `/skills` are modified and merged, production artifacts in `/dist` are compiled automatically via automation workflows to maintain absolute parity.*

---

## Production-Ready Skills Matrix

* **`branch-onboarding-expert`** Establishes strict guardrails for handling Branch worker onboarding and payout flows. Strategically guides developer models away from high-risk or legacy endpoints that mishandle PII/KYC data, forcing alignment with Branch's secure hosted onboarding screens.
  
* **`branch-integration-audit`** An autonomous code review engine. When fed active integration files or code snippets, it runs a structured compliance check, generating a definitive report highlighting structural vulnerabilities, integration warnings, and passing architectural patterns.

---

## Core Client Compatibility

The skills provided in this hub are optimized for the `SKILL.md` syntax native to Anthropic's Claude ecosystem. While the underlying functional logic and compliance guidelines remain universally applicable across any generative AI framework, the explicit setup, ingestion, and execution instructions detailed above are tailored exclusively to Claude workspaces.
