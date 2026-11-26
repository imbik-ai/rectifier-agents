# rectifier-agents

The Claude Code plugin for [Rectifier AI code review](https://rectifier.imbik.ai/), imbik's code review agent. The plugin adds:

- **The rectifier skill**, which teaches the agent to review a branch, commits or uncommitted changes with the
  `rectifier` CLI, and to read a review's state and findings.
- **Rectifier's MCP server**, `https://mcp.rectifier.imbik.ai/mcp`, which lets the agent read your account's reviews:
  which reviews ran, how each one went and what it found. The server only reads, and every call needs a sign-in.
  It is listed in the [official MCP registry](https://registry.modelcontextprotocol.io/) as `ai.imbik/rectifier`, and on
  [Glama](https://glama.ai/mcp/connectors/ai.imbik/rectifier).
- **Hooks** that send the Claude Code session behind a pull request to Rectifier. They send nothing until you run
  `rectifier transcripts enable` on the machine.

## Install

Install the CLI, then add the plugin in Claude Code:

```sh
uv tool install imbik-rectifier     # or: pipx install imbik-rectifier
claude plugin marketplace add imbik-ai/rectifier-agents
claude plugin install rectifier@rectifier
```

To sign in to the MCP server, type `/mcp` in Claude Code, pick rectifier and choose Authenticate. You need a Rectifier
account that has accepted the Terms of Service.

Other agents can use the MCP server without the plugin. Codex:
`codex mcp add rectifier --url https://mcp.rectifier.imbik.ai/mcp`, then `codex mcp login rectifier`. Cursor: the entry
on the dashboard's Connect an agent page.

## Docs

- CLI and skill: https://app.rectifier.imbik.ai/docs/cli
- MCP server: https://app.rectifier.imbik.ai/docs/mcp

## Layout

- `.claude-plugin/marketplace.json`: the marketplace, with one plugin.
- `claude-code/rectifier/`: the plugin. Its version follows the `imbik-rectifier` CLI. Each CLI release copies it here
  and replaces what is there, so a change made here directly is lost at the next release.

## License

Apache License 2.0, in [LICENSE](LICENSE). It covers this repository. The `imbik-rectifier` CLI and the Rectifier
service have their own terms.
