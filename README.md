# rgl-rails

Rails 8 + Full Hotwire Stack skills plugin for Claude Code by [Ruby Growth Labs](https://rubygrowthlabs.com).

## What's Included

7 domain-scoped skills covering the complete Rails + Hotwire stack:

| Skill | Description |
|-------|-------------|
| **rails-architecture** | DHH-style architecture, fat models, thin controllers, REST mapping, Minitest |
| **turbo-navigation** | Turbo Drive caching, Turbo Frames for tabs, pagination, lazy loading, faceted search |
| **turbo-streams** | Real-time updates, broadcasting, custom actions, Turbo 8 morphing, optimistic UI |
| **stimulus-controllers** | Lifecycle, values, targets, outlets, action parameters, production controllers |
| **forms-validation** | Inline editing, modal forms, typeahead, external form controls, submission lifecycle |
| **hotwire-native** | Turbo iOS/Android, path configuration, Strada bridge components, native auth |
| **frontend-craft** | CSS @layer, loading indicators, view transitions, optimistic morphing, frame spinners |

## Two-Layer Knowledge Architecture

Each skill contains two complementary knowledge layers:

- **`references/`** — Synthesized, opinionated guidance: pattern cards, guardrails, DHH-style recommendations ("what to do and why")
- **`handbook/`** — Official Turbo, Stimulus, and Hotwire Native documentation: precise API specs, attribute names, event names, action descriptors ("what exists and how it works")

The `references/` layer tells you the right pattern. The `handbook/` layer gives you the exact API details to implement it correctly.

## Requirements

- **Claude Code** (CLI) — [install instructions](https://code.claude.com/docs/en/quickstart)
- Not compatible with Claude Desktop (which uses MCP servers, not skills)

## Installation

### From Marketplace (Recommended)

Add the rgl-rails marketplace, then install the plugins you want:

```
/plugin marketplace add rubygrowthlabs/rgl-rails
```

Then install all 7 plugins:

```
/plugin install rails-architecture@rgl-rails
/plugin install turbo-navigation@rgl-rails
/plugin install turbo-streams@rgl-rails
/plugin install stimulus-controllers@rgl-rails
/plugin install forms-validation@rgl-rails
/plugin install hotwire-native@rgl-rails
/plugin install frontend-craft@rgl-rails
```

Or browse and install interactively — run `/plugin` and go to the **Discover** tab.

### Team/Project Setup

Add this to your project's `.claude/settings.json` so teammates get the marketplace automatically:

```json
{
  "extraKnownMarketplaces": {
    "rgl-rails": {
      "source": {
        "source": "github",
        "repo": "rubygrowthlabs/rgl-rails"
      }
    }
  }
}
```

### Local Development / Testing

```bash
claude --plugin-dir ./path/to/rgl-rails
```

Load a local copy for development or testing changes before publishing.

## Usage

Skills auto-activate when your conversation mentions relevant keywords — no manual setup needed.

### Slash Commands

- `/rails-architecture:review` — Review code against DHH/Rails conventions
- `/rails-architecture:analyze` — Scan codebase for simplification opportunities
- `/rails-architecture:simplify [goal]` — Plan incremental refactoring toward vanilla Rails

### Verify Installation

Run `/plugin` in Claude Code and go to the **Installed** tab to confirm the plugins are loaded.

## Trigger Phrases

Skills auto-activate on relevant keywords:

- **rails-architecture**: "controller", "concern", "vanilla rails", "dhh style", "minitest", "fixture"
- **turbo-navigation**: "turbo frame", "lazy load", "pagination", "turbo drive", "cache", "tabs"
- **turbo-streams**: "turbo stream", "broadcast", "real-time", "morph", "custom action"
- **stimulus-controllers**: "stimulus", "controller", "values", "targets", "outlets"
- **forms-validation**: "form", "validation", "inline edit", "typeahead", "modal form"
- **hotwire-native**: "turbo native", "ios", "android", "strada", "bridge component", "mobile"
- **frontend-craft**: "css", "loading", "spinner", "view transition", "animation"

## License

MIT
