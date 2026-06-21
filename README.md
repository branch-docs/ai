# Branch AI & MCP Gateway

Welcome to Branch's centralized hub for public AI resources and integrations. This repository bridges the gap between AI assistants and the Branch platform by exposing two distinct integration vectors:

1. **Automated MCP Infrastructure:** A live, cloud-hosted Model Context Protocol (MCP) server for real-time API reference discovery.
2. **Contextual Claude Skills:** Specialized, pre-configured instruction sets that guide AI models through compliant engineering and onboarding patterns.

---

## 1. Automated Model Context Protocol (MCP) Server

Branch runs a managed cloud-native MCP server directly through our core documentation platform. This server securely exposes read-only tools allowing AI agents to dynamically query, search, and parse the live Branch API reference. 

Because this is hosted via the cloud, there is no local environment configuration required—no local runtime dependencies, Node modules, scripts, or manual compilation.

* **Endpoint Gateway URL:** `https://docs.branchapp.com/mcp`
* **Transport Mechanism:** Server-Sent Events (SSE)

### Integration & Configuration

To expose Branch's documentation tools to an MCP-capable client (such as Claude Desktop), add the endpoint to your ecosystem configuration file. 

For **Claude Desktop**, append the following block to your configuration schema (`~/.config/Claude/claude_desktop_config.json` on macOS, or `%APPDATA%\Claude\claude_desktop_config.json` on Windows) and restart the application:

```json
{
  "mcpServers": {
    "branch-docs": {
      "type": "sse",
      "url": "[https://docs.branchapp.com/mcp](https://docs.branchapp.com/mcp)"
    }
  }
}
