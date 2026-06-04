# Scrapfly Skills

Agent skills for web scraping, extraction, screenshots, crawling, browser
automation, the [`scrapfly` CLI](https://github.com/scrapfly/scrapfly-cli),
and the Scrapfly MCP toolset.

Compatible with Claude Code, Cursor, Cline, Codex, Windsurf, Gemini, GitHub
Copilot, Roo, Goose, Antigravity, and [50+ other agents](https://skills.sh)
via the open [`skills`](https://www.npmjs.com/package/skills) CLI.

## Install

Pick the flow that matches what you're shipping:

```bash
# Install every Scrapfly skill into every detected agent
npx skills add scrapfly/skills --all

# Pick specific skills (use the canonical `--skill <name>` form)
npx skills add scrapfly/skills --skill scrapfly-scraper
npx skills add scrapfly/skills --skill scrapfly-cli --skill scrapfly-agent-rules

# Target specific agents (claude-code, codex, cursor, gemini-cli, ...)
npx skills add scrapfly/skills --all -a claude-code -a cursor

# Install globally (~/<agent>/skills/) instead of in the current project
npx skills add scrapfly/skills --all --global

# Just list what's available without installing
npx skills add scrapfly/skills --list
```

`scrapfly/skills` is GitHub-shorthand for
[`https://github.com/scrapfly/skills`](https://github.com/scrapfly/skills);
the CLI clones it directly — no Scrapfly account or registry call required.

## Skills

| Skill | When to use |
|-------|-------------|
| **scrapfly-scraper** | Python SDK agent calling `scrapfly-sdk` for scraping with proxy rotation, anti-bot bypass, and JS rendering |
| **scrapfly-extraction** | Extract structured data from HTML/markdown using LLM prompts, AI models, or templates |
| **scrapfly-screenshot** | Capture web page screenshots with JS rendering and anti-bot bypass |
| **scrapfly-crawler** | Crawl entire websites with link discovery, depth control, and data extraction |
| **scrapfly-browser** | Automate cloud browsers via Playwright with proxy rotation and fingerprinting |
| **scrapfly-cli** | Shell-tool agents (Claude Code, Codex, Aider) driving the [`scrapfly`](https://github.com/scrapfly/scrapfly-cli) CLI binary instead of writing Python |
| **scrapfly-agent-rules** | Cross-tool golden rules for an LLM agent connected to the Scrapfly MCP server (stateless vs. browser, when to `unblock`, WebMCP) |
| **scrapfly-sdk-typescript** | Web scraping from TypeScript/JavaScript (Node.js, Deno, Bun) with the `scrapfly-sdk` package |
| **scrapfly-sdk-go** | Web scraping from Go with the `go-scrapfly` module and its built-in goquery selector |
| **scrapfly-sdk-rust** | Web scraping from Rust with the async `scrapfly-sdk` crate |

## Setup

For the Python SDK skills (`scrapfly-scraper`, `scrapfly-extraction`,
`scrapfly-screenshot`, `scrapfly-crawler`, `scrapfly-browser`):

```bash
pip install scrapfly-sdk
export SCRAPFLY_API_KEY="scp-live-..."
```

For the other-language SDK skills, install the matching package:

```bash
npm install scrapfly-sdk                     # scrapfly-sdk-typescript
go get github.com/scrapfly/go-scrapfly       # scrapfly-sdk-go
cargo add scrapfly-sdk                       # scrapfly-sdk-rust
export SCRAPFLY_API_KEY="scp-live-..."
```

For the CLI skill (`scrapfly-cli`):

```bash
curl -fsSL https://scrapfly.io/scrapfly-cli/install | sh
export SCRAPFLY_API_KEY="scp-live-..."
```

For the agent-rules skill (`scrapfly-agent-rules`): nothing to install on the
agent side — the rules assume your agent is connected to a Scrapfly MCP
server. See [agent-ai](https://github.com/scrapfly/agent-ai) for reference
ADK bootstraps in Python and Go.

Get your API key at [scrapfly.io](https://scrapfly.io).

## Updating & removing

```bash
# Pull the latest version of every installed Scrapfly skill
npx skills update

# Remove specific skills
npx skills remove scrapfly-scraper

# Remove every Scrapfly skill from a specific agent
npx skills remove --skill 'scrapfly-*' -a claude-code
```

## Where skills land

`npx skills` symlinks the skills into each agent's conventional directory.
A few examples:

| Agent | Project install | Global install |
|-------|-----------------|----------------|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Codex | `.agents/skills/` | `~/.codex/skills/` |
| Cursor | `.agents/skills/` | `~/.cursor/skills/` |
| Gemini CLI | `.agents/skills/` | `~/.gemini/skills/` |
| OpenCode | `.agents/skills/` | `~/.config/opencode/skills/` |

Full agent matrix: see [`skills` package README](https://www.npmjs.com/package/skills).
