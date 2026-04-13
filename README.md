# AI Skills

Open-source skills for AI coding agents. Currently in [Claude Code](https://claude.ai/code) SKILL.md format — contributions for other platforms welcome.

By [Human Pages AI](https://humanpages.ai).

## Skills

| Skill | Description |
|-------|-------------|
| [war](./war) | Multi-perspective adversarial debate. Spawns 6 sub-agents with opposing viewpoints across 3 rounds to stress-test any plan, strategy, or decision. |
| [audit-contract](./audit-contract) | Adversarial smart contract security audit. Auto-selects 5-7 specialist agents (from a roster of 11) based on contract features. Runs Slither, writes Foundry PoC tests, produces a ranked finding list with severity and code fixes. |
| [screenshot](./screenshot) | Reads and analyzes the latest screenshot from your Screenshots folder. |
| [humanpages](./humanpages) | Hire real humans for tasks agents can't do alone — QA testing, directory submissions, translations, and more. 36 MCP tools for the full hiring lifecycle. |

## Install

### Claude Code

Clone the repo and add individual skills:

```bash
git clone https://github.com/human-pages-ai/ai-skills.git
```

Then add whichever skills you want:

```bash
claude skills add /path/to/ai-skills/war
claude skills add /path/to/ai-skills/screenshot
```

For **humanpages**, the MCP server is the recommended install:

```bash
claude mcp add humanpages -- npx -y humanpages
```

The skill file in this repo documents the full workflow and is useful as a reference even if you use the MCP server directly.

### Other Platforms

These skills are markdown files describing agent behavior. The concepts (adversarial debate, screenshot analysis, human hiring) are platform-agnostic — adapt the prompts to your tool's format. PRs for Cursor, Windsurf, Copilot, or other formats are welcome.

## Contributing

PRs welcome — new skills, improvements to existing ones, or ports to other platforms.

Please keep skills self-contained (one folder, one SKILL.md) and include a clear description in the YAML frontmatter.

## License

MIT
