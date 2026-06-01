

# Branch AI & MCP Hub

Welcome to Branch's official public repository for AI resources, developer context, and Model Context Protocol (MCP) assets. This repository serves as a foundational knowledge layer for LLMs, autonomous agents, and internal tooling interacting with the Branch ecosystem.

---

## ⚡️ Live Cloud MCP Server

Branch provides a fully managed, live cloud MCP server hosted directly through our developer documentation framework. Unlike standard local MCP setups, there is zero software compilation, Node dependency installation, or background scripting required on your machine.

* **Endpoint URL:** https://docs.branchapp.com/mcp
* **Transport Protocol:** Server-Sent Events (SSE)

---

## 🛠️ Claude Desktop Configuration

To instantly connect your Claude Desktop client to the Branch live cloud server, add the following snippet to your configuration file (typically located at `~/.config/Claude/claude_desktop_config.json` on macOS or `%APPDATA%\Claude\claude_desktop_config.json` on Windows):

```json
{
  "mcpServers": {
    "branch": {
      "type": "sse",
      "url": "https://docs.branchapp.com/mcp"
    }
  }
}

```

Once added, restart Claude Desktop. The agent will automatically discover our active documentation and discovery tools, allowing it to navigate, search, and parse Branch API paths dynamically.

---

## 🧠 Pre-Baked Agent Skills

Beyond raw API schemas, this repository maintains platform-agnostic behavioral frameworks to align AI models with specific corporate operational standards.

Browse the `/skills` directory to explore copy-pasteable system prompts, persona blueprints, and compliance guardrails structured as individual `SKILL.md` directives:

* **branch-onboarding-expert**: Guardrails, references, and workflows tailored for optimizing Customer Service agent onboarding and platform navigation.
* **branch-integration-audit**: Instructions for validating implementation patterns, API key enforcement, and developer compliance baselines.

> **Note on Compatibility:** While formatted using markdown conventions optimal for Anthropic's Claude ecosystem, these directives are platform-agnostic and can be utilized across OpenAI Assistants, Microsoft Copilot Studio, or custom orchestration layers like LangChain.

