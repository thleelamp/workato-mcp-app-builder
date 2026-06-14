# Workato MCP App Builder (Claude skill)

A Claude [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that generates the code for a [Workato MCP App](https://docs.workato.com/en/mcp/mcp-apps) from a tool's input and output schema. You give it the tool name and the two schemas, and it produces a single self-contained HTML file ready to paste into the Workato MCP App code editor.

## Why this exists

The Workato MCP App editor today gives you two starting points: **Start from scratch** (hand-write the code) or a generic **Submission form** template that is not tied to your tool's schema. Neither produces an App tailored to a specific tool. This skill fills that gap, so people who cannot write JavaScript can still ship a working, schema-accurate App from three pasted fields and edit it lightly rather than build from zero.

## What it produces

A single `.html` file (HTML + Tailwind via CDN + one inline module script) that:

- builds an input form from the input schema,
- calls the linked tool via the `@modelcontextprotocol/ext-apps` SDK,
- renders the result as a summary, a sortable and filterable table, and an error banner, with a raw-response panel for debugging,
- adapts to whatever shape the tool actually returns (it does not hard-code field names).

It supports four patterns:

| Pattern | Use it for | Example |
| --- | --- | --- |
| Result-viewer (default) | Reads the user will browse: search, lookups, reports, dashboards | `mcp-app-builder/assets/example_app.html` |
| Write + refresh (POST then GET) | Acting on items: approve/reject, triage | `mcp-app-builder/assets/example_app_actions.html` |
| Form-first | True data-entry tools | `mcp-app-builder/assets/example_app.html` |
| Launcher + app-only worker | Opening the App without the model asking for parameters in chat | `mcp-app-builder/assets/example_app_launcher.html` |

## When NOT to use an MCP App

An App is only worth building when the interaction benefits from an interactive UI in the chat (exploring complex data, configuring many options, viewing rich media, real-time monitoring, or multi-step workflows). A **simple GET whose result can just be summarised in text should stay a plain tool response.** The skill enforces this check before generating anything.

## Quick start

1. Add the skill to Claude (see below).
2. Ask: *"Build an MCP app for my tool."*
3. Claude asks for three things: the **tool name**, the **input schema (JSON)**, and the **output schema (JSON)**. Paste them straight from your Workato tool configuration.
4. Claude returns one HTML file. Paste it into your Workato MCP server: **AI Hub > MCP servers > [your server] > Server capabilities > Apps > Add App > Start from scratch**, then link it to the tool.

## Tip: make the App open without chat questions

If the model keeps asking for a parameter in chat before the App opens, **mark the tool's input fields as optional.** With nothing required to chase, the model calls the tool immediately and the App opens; the form then collects the real input. For a guaranteed open, use the launcher + app-only worker pattern (a no-input launcher tool opens the App, and a hidden worker tool does the work on submit).

## Install the skill

This repo is the skill source. Pick whichever fits your setup:

- **Claude Project:** create a project and upload `mcp-app-builder/SKILL.md` plus the three files in `mcp-app-builder/assets/` as project knowledge. Anyone in the project can then ask Claude to build an App.
- **Claude Code / Cowork skills folder:** copy the `mcp-app-builder/` folder into your skills directory so the skill is discovered automatically.

## Repo structure

```
workato-mcp-app-builder/
├── README.md
├── LICENSE
├── .gitignore
└── mcp-app-builder/
    ├── SKILL.md                       # the skill
    └── assets/
        ├── example_app.html           # form-first search (generic renderer)
        ├── example_app_actions.html   # approve/reject (write + refresh)
        └── example_app_launcher.html  # launcher + app-only worker
```

## Known gaps and things to verify

This skill was built against the published Workato and MCP Apps docs, not a live runtime. Before relying on it, confirm the following in your environment:

- **SDK result shape.** The `ontoolresult` / `structuredContent` shape and `ontoolinput` availability are taken from the docs. Run a real tool and check the raw-response panel to confirm they match.
- **CDN import and CSP.** The templates load the SDK and Tailwind from jsdelivr. Confirm your MCP App's content security policy allows those origins (declare them under `_meta.ui.csp` `resourceDomains` if needed).
- **Host theming.** The templates ship a fixed light theme. They do not yet adapt to the host's light/dark theme via `McpUiHostContext` style variables.
- **Scale.** Filtering and sorting are client-side; there is no virtualisation or server-side pagination wiring for very large result sets.

## Contributing

Issues and pull requests are welcome, especially for host-theme adaptation, verified SDK result shapes, and additional patterns. Keep generated output to a single self-contained HTML file with no build step.

## References

- [Workato MCP Apps](https://docs.workato.com/en/mcp/mcp-apps)
- [Workato MCP server design best practices](https://docs.workato.com/en/mcp/mcp-server-design)
- [MCP Apps overview](https://modelcontextprotocol.io/extensions/apps/overview) and [ext-apps API](https://apps.extensions.modelcontextprotocol.io/api/)

## License

[MIT](LICENSE)
