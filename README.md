# Branch AI & MCP Hub

This repository holds Branch's public AI resources: the address of our Model
Context Protocol (MCP) server, and a set of reusable Claude skills that help AI
assistants work with the Branch platform correctly.

There are two separate things here, and they work in different ways. Read the
section that matches what you're trying to do.

---

## 1. The MCP Server (live tools)

Branch runs a managed cloud MCP server through our documentation platform. It
exposes read-only tools for searching and reading the Branch API docs. There's
nothing to install or run locally — no Node, no scripts, no compilation.

- **Endpoint URL:** `https://docs.branchapp.com/mcp`
- **Transport:** Server-Sent Events (SSE)

### Connecting it

In Claude (or another MCP-capable client), add a connector pointing at the
endpoint above. For Claude Desktop specifically, add this to your config file
(`~/.config/Claude/claude_desktop_config.json` on macOS, or
`%APPDATA%\Claude\claude_desktop_config.json` on Windows), then restart:

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

Once connected, the assistant can discover and use the server's documentation
tools automatically. This is the one part that *is* automatic — once the
connector is added, the tools show up on their own.

---

## 2. The Skills (manual install)

The `/skills` directory contains Claude skills — packaged instructions that
teach an assistant how to handle Branch integrations the right way (for example,
keeping KYC data out of the partner app and following the standard onboarding
and payout flow).

### Please read this first — skills do NOT load themselves

A skill in this repo will **not** be picked up by an AI assistant on its own.
This is the most common point of confusion, so to be explicit:

- An assistant will **not** discover these skills by browsing the repo.
- Asking an assistant to "go to this repo and load the skills" will **not**
  work — it can read the text of the files, but reading them is not the same as
  installing them, and they won't persist or activate.
- Connecting the MCP server (section 1) does **not** bring the skills along.
  The connector carries the server's tools, not these skills. They are
  unrelated mechanisms.

To use a skill, a person has to install it into their own Claude. Here's how.

### Installing a skill (individual)

1. Go to the `/dist` folder and download the `.zip` for the skill you want
   (see "What's in /dist" below).
2. In Claude, open **Customize → Skills**.
3. Add the skill by uploading the `.zip` file.

A skill you add this way is private to your own account. (Code execution must be
enabled in your Claude settings for skills to run.)

### Making a skill available to everyone (organizations)

If you want a skill available to your whole organization so individuals don't
each have to install it, an **organization owner** provisions it once via
**Organization settings → Skills**, using the same `.zip` from `/dist`. It then
appears in everyone's skill list automatically. This is an internal admin action
on your side — it isn't something this repo does or that a reader needs to set
up themselves.

---

## What's in `/skills` vs `/dist`

This repo keeps two copies of each skill, for two different purposes:

- **`/skills`** — the **source**. These are the editable, human-readable files
  (`SKILL.md` and any reference docs), kept in folders so they're easy to read,
  review, and update in GitHub. This is where changes get made. Read these if
  you want to see what a skill actually does, or contribute a change.

- **`/dist`** — the **ready-to-install packages**. Each skill folder from
  `/skills` is bundled into a single `.zip` file here. These are what you
  download and upload into Claude. They contain exactly the same content as the
  source folders, just packaged into one file so the upload works in a single
  step (and so any reference files stay attached correctly). If you just want to
  *use* a skill, `/dist` is the only folder you need.

When a skill in `/skills` changes, its `.zip` in `/dist` is rebuilt to match.

---

## Available Skills

- **branch-onboarding-expert** — the standard, correct path for Branch worker
  onboarding and payouts. Steers integrations away from API endpoints that look
  usable but mishandle sensitive data, and toward Branch's hosted onboarding.
- **branch-integration-audit** — reviews existing Branch integration code and
  produces a structured report flagging violations, warnings, and passing
  checks.

---

## A note on compatibility

These skills are written in the Markdown/`SKILL.md` format used by Claude. The
underlying guidance (the correct integration path, what to avoid) is general and
can inform work in other AI tools too, but the install steps above are specific
to Claude. Other platforms have their own mechanisms for adding this kind of
guidance.
