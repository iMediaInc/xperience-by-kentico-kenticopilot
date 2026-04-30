# Usage guide

This guide explains how to use the AI agent skills in this repository for Xperience by Kentico development.

## Prerequisites

Before you start, you need:

- An AI coding assistant (e.g., [GitHub Copilot](https://github.com/features/copilot), [Claude Code](https://www.claude.com/product/claude-code), [Cursor](https://cursor.com))
- Git installed on your machine
- Access to an Xperience by Kentico project

## Install an AI coding assistant

Install an AI coding assistant. This repository provides plugins and skills tested with select popular solutions. To transfer the skills to other assistants, follow the conventions of your specific solution.

### GitHub Copilot

1. Install the [GitHub Copilot extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot) for VS Code.
2. Sign in to your GitHub account.

### Claude Code

1. Install [Claude Code](https://www.claude.com/product/claude-code).
2. Sign in to your Anthropic account.

### Cursor

1. Install [Cursor](https://cursor.com).
2. Sign in as required for your Cursor plan.

## Install plugins from the marketplace

This repository provides marketplace manifests for VS Code (GitHub Copilot), Claude Code, and Cursor. Install plugins from the respective marketplace or team import when configured.

### VS Code (GitHub Copilot)

1. Add the marketplace to your VS Code settings (`settings.json`):

   ```json
   "chat.plugins.marketplaces": [
       "Kentico/xperience-by-kentico-kenticopilot"
   ]
   ```

2. Open the Extensions sidebar and search `@agentPlugins` to browse available plugins.
3. Select **Install** on the plugin you need.

### Copilot CLI

```bash
copilot plugin marketplace add Kentico/xperience-by-kentico-kenticopilot
copilot plugin install widget-creation@xperience-by-kentico-kenticopilot
copilot plugin install kx13-codebase-migration@xperience-by-kentico-kenticopilot
```

### Claude Code

```bash
/plugin marketplace add Kentico/xperience-by-kentico-kenticopilot
/plugin install widget-creation@xperience-by-kentico-kenticopilot
/plugin install kx13-codebase-migration@xperience-by-kentico-kenticopilot
```

### Install in Cursor

1. Read [Plugins](https://cursor.com/docs/plugins) for an overview of plugin contents (skills, MCP, rules, and so on).
2. **Team marketplace:** If your organization uses Cursor Teams or Enterprise, an admin can import this GitHub repository under **Dashboard → Settings → Plugins** ([team marketplaces](https://cursor.com/docs/plugins#team-marketplaces)). Members install plugins from the marketplace panel in Cursor.
3. **Public marketplace:** Submit the repository at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) for broad distribution after Cursor review.
4. **Local testing:** Copy or symlink a plugin folder (for example `plugins/widget-creation` from a clone of this repo) into `%USERPROFILE%\.cursor\plugins\local\<plugin-name>` on Windows or `~/.cursor/plugins/local/<plugin-name>` on macOS/Linux. The plugin root must include `.cursor-plugin/plugin.json`. Reload the window and verify skills and MCP in Cursor Settings. See [Test plugins locally](https://cursor.com/docs/plugins#creating-plugins).

## Copy plugin files manually (alternative)

If you prefer not to use the plugin marketplace, copy plugin files directly into your project workspace.

1. Clone this repository:

   ```bash
   git clone https://github.com/Kentico/xperience-by-kentico-kenticopilot.git
   ```

2. Copy the plugin folder for your use case to the root of your project workspace:

   ```bash
   # Widget creation
   cp -r plugins/widget-creation/ YOUR_PROJECT/

   # KX13 codebase migration
   cp -r plugins/kx13-codebase-migration/ YOUR_PROJECT/
   ```

## Use the skills

After installing a plugin (or copying the files manually):

1. Open your project in your AI coding assistant.
2. Open the README for the plugin you installed (e.g., `plugins/widget-creation/README.md`).
3. Follow the instructions in the README.

Each plugin README explains:

- What the skills do
- How to trigger them
- What inputs are expected
- Example usage scenarios
