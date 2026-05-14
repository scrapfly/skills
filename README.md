# Scrapfly Skills

Agent skills for web scraping, extraction, screenshots, crawling, and browser automation using the [Scrapfly](https://scrapfly.io) API.

Compatible with Claude Code, Cursor, Cline, Windsurf, and [40+ other agents](https://skills.sh).

## Installation

Install all skills:

```bash
npx skills add scrapfly/skills
```

Or install individual skills:

```bash
npx skills add scrapfly/skills -s scrapfly-scraper
npx skills add scrapfly/skills -s scrapfly-extraction
npx skills add scrapfly/skills -s scrapfly-screenshot
npx skills add scrapfly/skills -s scrapfly-crawler
npx skills add scrapfly/skills -s scrapfly-browser
```

## Skills

| Skill | Description |
|-------|-------------|
| **scrapfly-scraper** | Web scraping with proxy rotation, anti-bot bypass, and JS rendering |
| **scrapfly-extraction** | Extract structured data from HTML/markdown using LLM prompts, AI models, or templates |
| **scrapfly-screenshot** | Capture web page screenshots with JS rendering and anti-bot bypass |
| **scrapfly-crawler** | Crawl entire websites with link discovery, depth control, and data extraction |
| **scrapfly-browser** | Automate cloud browsers via Playwright with proxy rotation and fingerprinting |

## Setup

```bash
pip install scrapfly-sdk
```

Set your API key:

```bash
export SCRAPFLY_API_KEY="your-api-key"
```

Get your API key at [scrapfly.io](https://scrapfly.io).

## Shell-tool agents (Claude Code, Codex, Aider, ...)

The skills above teach agents to write Python that calls the Scrapfly SDK.
If your agent runs shell commands instead of Python, install the CLI skill:

```bash
npx skills add scrapfly/scrapfly-cli
```

That ships a single `scrapfly-cli` skill plus the [`scrapfly` CLI](https://github.com/scrapfly/scrapfly-cli):
a JSON-on-stdout binary covering every Scrapfly product from one shell verb.
