# Monaiq AI Plugin

AI-powered software licensing and monetization assistant for [Monaiq](https://monaiq.com). This repository provides AI agent plugins for **GitHub Copilot (VS Code)**, **Claude Code**, and **OpenAI Codex** — giving each platform a specialized Monaiq agent with skills, tools, and domain knowledge.

## What is Monaiq?

Monaiq is a licensing and monetization platform that helps product creators:

- **Discover** what capabilities in their codebase are worth licensing
- **Design** pricing strategies and monetization models
- **Build** product catalogs with features, offerings, and billing
- **Integrate** the Monaiq SDK into .NET and React applications
- **Troubleshoot** licensing and SDK integration issues

The Monaiq AI agent guides you through the entire journey — from analyzing your codebase to implementing license enforcement in your application.

## Plugins

This repository contains three platform-specific plugins, each tailored to the conventions and capabilities of its host AI tool:

| Plugin | Platform | Directory |
|--------|----------|-----------|
| **Copilot** | GitHub Copilot in VS Code | [`copilot/`](copilot/) |
| **Claude Code** | Claude Code (terminal & IDE) | [`claude-code/`](claude-code/) |
| **Codex** | OpenAI Codex (ChatGPT & CLI) | [`codex/`](codex/) |

All three plugins provide the same Monaiq agent with identical domain knowledge, MCP tools, and skill routing. The differences are in file format and installation method.

---

## Installation

### GitHub Copilot (VS Code)

The Copilot plugin uses VS Code's [Agent Plugins](https://code.visualstudio.com/docs/copilot/customization/agent-plugins) system (preview).

> **Prerequisite:** VS Code with GitHub Copilot extension installed. Enable agent plugins via the `chat.plugins.enabled` setting.

#### Option A — Install from source (recommended)

1. Open the **Command Palette** (`Ctrl+Shift+P`) and run **Chat: Install Plugin From Source**.
2. Enter this repository's URL:
   ```
   https://github.com/Sidub-Inc/Monaiq.AI.Plugin
   ```
3. VS Code clones the plugin and registers the `copilot/` directory. The Monaiq agent, skills, and MCP tools become available in chat automatically.

#### Option B — Clone locally

1. Clone this repository:
   ```bash
   git clone https://github.com/Sidub-Inc/Monaiq.AI.Plugin.git
   ```
2. Register the plugin directory in your VS Code `settings.json`:
   ```jsonc
   // settings.json
   "chat.pluginLocations": {
       "C:/path/to/Monaiq.AI.Plugin/copilot": true
   }
   ```
3. Reload VS Code. The plugin appears in the **Agent Plugins — Installed** section of the Extensions view.

#### Verify installation

- Open the Extensions sidebar (`Ctrl+Shift+X`) and search `@agentPlugins` to confirm the Monaiq plugin is listed.
- In Chat, the `@monaiq` agent and its skills should be available.

#### Plugin contents

| Path | Description |
|------|-------------|
| [`copilot/plugin.json`](copilot/plugin.json) | Plugin metadata and configuration |
| [`copilot/agents/monaiq.agent.md`](copilot/agents/monaiq.agent.md) | Monaiq agent definition (persona, tools, modes) |
| [`copilot/copilot-instructions.md`](copilot/copilot-instructions.md) | Shared domain knowledge and skill architecture |
| [`copilot/instructions/`](copilot/instructions/) | Context-specific instruction files |
| [`copilot/skills/`](copilot/skills/) | Agent skills (getting-started, troubleshoot-integration) |

---

### Claude Code

The Claude Code plugin uses [AGENTS.md](https://code.claude.com/docs/en/memory) files and the [plugin system](https://code.claude.com/docs/en/plugins).

#### Option A — Project-level installation

Copy the `claude-code/` directory contents into your project's `.claude/` directory:

```bash
# From your project root
git clone https://github.com/Sidub-Inc/Monaiq.AI.Plugin.git /tmp/monaiq-plugin
cp -r /tmp/monaiq-plugin/claude-code/AGENTS.md ./AGENTS.md
cp -r /tmp/monaiq-plugin/claude-code/agents ./.claude/agents
cp -r /tmp/monaiq-plugin/claude-code/skills ./.claude/skills
```

The `AGENTS.md` file at the project root provides domain knowledge to Claude Code automatically. The agent is available as a subagent via `/agent monaiq` or is invoked automatically based on context.

#### Option B — User-level installation

To make the Monaiq agent available across all projects, copy the agent file to your user-level agents directory:

```bash
cp claude-code/agents/monaiq.md ~/.claude/agents/monaiq.md
```

#### Option C — Plugin marketplace

If your organization uses a Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces), the Monaiq plugin can be added as a marketplace entry pointing to this repository. Add the marketplace to your settings:

```jsonc
// .claude/settings.json
{
  "extraKnownMarketplaces": {
    "monaiq": {
      "source": {
        "source": "github",
        "repo": "Sidub-Inc/Monaiq.AI.Plugin"
      }
    }
  }
}
```

Then install via `/plugin install monaiq` in Claude Code.

#### Plugin contents

| Path | Description |
|------|-------------|
| [`claude-code/AGENTS.md`](claude-code/AGENTS.md) | Shared domain knowledge, tool routing, and skill architecture |
| [`claude-code/agents/monaiq.md`](claude-code/agents/monaiq.md) | Monaiq subagent definition (persona, tools, modes) |
| [`claude-code/skills/`](claude-code/skills/) | Skills organized by directory (getting-started, manage-catalog, etc.) |

---

### OpenAI Codex

The Codex plugin uses [AGENTS.md](https://openai.com/index/introducing-codex/) files, which Codex reads automatically from your repository.

#### For Codex Web (ChatGPT)

1. Connect your GitHub repository to Codex at [chatgpt.com/codex](https://chatgpt.com/codex).
2. Copy the `codex/AGENTS.md` file to the root of your repository:
   ```bash
   cp codex/AGENTS.md ./AGENTS.md
   ```
3. Copy the skills directory:
   ```bash
   cp -r codex/skills ./.codex/skills
   ```
4. Commit and push. Codex automatically reads the `AGENTS.md` file when it clones your repository for task execution.

#### For Codex CLI

1. Install Codex CLI:
   ```bash
   npm install -g @openai/codex
   # or
   brew install --cask codex
   ```
2. Place the `AGENTS.md` file in your project root (same as for Codex Web above).
3. Run `codex` in your project directory. The agent reads `AGENTS.md` on startup.

#### Plugin contents

| Path | Description |
|------|-------------|
| [`codex/AGENTS.md`](codex/AGENTS.md) | Shared domain knowledge, tool routing, and skill architecture |
| [`codex/skills/`](codex/skills/) | Skills organized by directory (getting-started, manage-catalog, etc.) |

---

## Usage

Once installed, the Monaiq agent is available in your AI tool's chat interface. The agent operates in two modes:

### Discovery Mode

Analyze your codebase, evaluate licensing scenarios, and design pricing strategies. Triggered by phrases like:

- *"Analyze my app for licensable features"*
- *"What pricing model should I use?"*
- *"Compare subscription vs. perpetual licensing for my product"*

### Implementation Mode

Create products, configure offerings, integrate the SDK, and manage your catalog. Triggered by phrases like:

- *"Create a new product with a trial offering"*
- *"Set up the Monaiq SDK in my .NET project"*
- *"Add a rate-limited API feature to my product"*

### Getting Started

The recommended entry point is the **getting-started** skill. Simply ask:

> *"Help me get started with Monaiq"*

The agent detects your current state (new user, partially configured, or returning) and routes you to the appropriate next step:

1. **Onboarding** — Account setup, profile configuration, credential retrieval
2. **Catalog** — Product creation, feature definition, offering configuration
3. **Integration** — SDK setup, feature enforcement, purchase flow implementation

### Troubleshooting

If you encounter issues with the SDK or integration, ask:

> *"Help me troubleshoot my Monaiq integration"*

The agent uses structured diagnostic decision trees to walk through setup, authentication, validation, and consumption errors.

### Check Progress

Ask *"What's my current status?"* or *"Check my progress"* at any time to see where you are in the Monaiq journey.

---

## MCP Tools & Resources

All plugins connect to the same Monaiq MCP server, providing:

| Category | Tools | Purpose |
|----------|-------|---------|
| **Session** | `register_or_login` | Authentication |
| **Onboarding** | `getting_started`, `profile` | Account setup and orientation |
| **Catalog** | `product`, `product_feature`, `offering`, `feature_offering` | Product lifecycle management |
| **Integration** | `implement_base`, `implement_feature`, `implement_purchase_flow` | SDK integration guidance |

Resources available via `monaiq://` URIs include domain models, namespace mappings, SDK setup guides, licensing scenario patterns, pricing patterns, and troubleshooting decision trees.

---

## License

[MIT](LICENSE)

## Links

- [Monaiq Platform](https://monaiq.com)
- [Monaiq Documentation](https://docs.monaiq.com)
- [Sidub Inc.](https://github.com/Sidub-Inc)
