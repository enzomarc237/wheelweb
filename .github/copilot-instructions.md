# Copilot Instructions for wheelweb

This project uses the BMAD-METHOD agent system and OpenCode for multi-agent, workflow-driven development. AI agents must follow these conventions to be productive and avoid common pitfalls.

## Big Picture Architecture
- Agents and teams are defined in `AGENTS.md` and `web-bundles/agents/`, with expansion packs in `web-bundles/expansion-packs/`.
- Each agent (e.g., dev, qa, po) has a persona file with activation, workflow, and interaction rules. See `.clinerules/` and `.cursor/rules/bmad/` for agent-specific standards.
- Core workflows and tasks are managed via `.bmad-core/` (not included here, but referenced in agent rules).
- The system is designed for role-based, task-driven development. Always activate the correct agent persona before starting work.

## Developer Workflows
- **Activation:** Always read the full agent rule file before activating a persona (see `.clinerules/08-dev.md` for dev). Greet the user and run `*help` to show available commands.
- **Task Execution:** Only load dependency files when requested by the user or when executing a specific task. Do not pre-load unrelated files.
- **Story/Task Flow:** Do not begin development until a story is out of draft mode and explicit permission is given.
- **Interaction:** Tasks with `elicit=true` require strict user interaction—never skip required prompts for efficiency.
- **Options Presentation:** When listing tasks/templates, always present as a numbered list for user selection.

## Project-Specific Conventions
- **Dependency Resolution:** Dependencies map to `.bmad-core/{type}/{name}`. Only load these when executing commands that reference them.
- **Customization Precedence:** The `agent.customization` field in agent files overrides any conflicting instructions.
- **Workflow Precedence:** Formal task instructions always override base behavioral constraints.
- **File Loading:** Only load files listed in `devLoadAlwaysFiles` or those explicitly requested.

## Integration Points
- Expansion packs add new agents and teams in `web-bundles/expansion-packs/`.
- All agents/tools are enabled for write, edit, and bash operations.
- Orchestrators run in primary mode; other agents run as all.

## Examples
- To activate the dev agent: read `.clinerules/08-dev.md`, greet, run `*help`, then await user command.
- To execute a story task: ensure story is not in draft, follow task instructions exactly, interact as required.
- To resolve a dependency: map the request to `.bmad-core/{type}/{name}` and load only when needed.

## Key Files & Directories
- `AGENTS.md`: Agent directory and usage guidance
- `.clinerules/`: Agent rules and standards
- `.cursor/rules/bmad/`: Additional agent rules
- `web-bundles/agents/`: Agent persona definitions
- `web-bundles/expansion-packs/`: Optional agent/team packs

---

If any section is unclear or missing critical workflow details, please provide feedback for further refinement.