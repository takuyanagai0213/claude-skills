# Claude Code Skills

A collection of reusable skills for [Claude Code](https://claude.com/claude-code).

## What are Skills?

Skills are workflow guides that provide step-by-step instructions for common tasks. They automatically activate when you request help with specific types of work.

## Available Skills

### [Incident Response Guide](./incident-response-guide/)

A systematic workflow for investigating production alerts and incidents using GCP observability tools.

**When to use:**
- "Investigate this alert"
- "Troubleshoot this incident"
- "アラート対応して"

**Features:**
- Read-only investigation workflow (no resource modifications)
- Integration with GCP MCP tools (Cloud Logging, Monitoring, Trace)
- Step-by-step investigation checklist
- Incident report template

## Installation

1. Copy the skill directory to your project's `.claude/skills/` folder:

```bash
# Clone this repository
git clone https://github.com/takuyanagai0213/claude-skills.git

# Copy a skill to your project
cp -r claude-skills/incident-response-guide /path/to/your/project/.claude/skills/
```

2. Customize the skill for your project (if needed):
   - Edit `project-config.example.md` with your environment details
   - Update service names and project IDs as needed

3. Restart Claude Code or start a new conversation to activate the skill

## Customization

Each skill may include a `project-config.example.md` file with project-specific information. Copy and customize this file for your own environment.

## Contributing

Contributions are welcome! Please ensure skills:
- Are generic and reusable across different projects
- Include clear documentation and examples
- Follow the established directory structure
- Remove project-specific details from the main skill file

## License

MIT License - See LICENSE file for details

## Skills Included

- **incident-response-guide** - GCP incident investigation workflow
- (More skills coming soon)
