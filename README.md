# Walmart APIs SDK Plugin

A plugin whose skills teach a coding agent to install and use the APIMatic-generated **Walmart APIs SDK**, in C#/.NET, Python, TypeScript. Every SDK fact the skills state is grounded in the SDK's own source and generated documentation, not in what a model remembers about this API.

## What's inside

One skill set per language. The entry point is that language's getting-started skill, which carries what is specific to this SDK; the rest are API-agnostic and describe how to use any SDK the same generator produces.

| Language | Skill prefix | Skills |
| --- | --- | --- |
| C#/.NET | `dotnet-` | `dotnet-authentication`, `dotnet-calling-endpoints`, `dotnet-client-initialization`, `dotnet-configuration-resilience`, `dotnet-error-handling`, `dotnet-getting-started`, `dotnet-integrate-walmart-apis`, `dotnet-models`, `dotnet-testing` |
| Python | `python-` | `python-authentication`, `python-calling-endpoints`, `python-client-initialization`, `python-configuration-resilience`, `python-error-handling`, `python-event-streams`, `python-file-handling`, `python-getting-started`, `python-integrate-walmart-apis`, `python-models`, `python-testing` |
| TypeScript | `typescript-` | `typescript-authentication`, `typescript-calling-endpoints`, `typescript-client-initialization`, `typescript-configuration-resilience`, `typescript-error-handling`, `typescript-event-streams`, `typescript-getting-started`, `typescript-integrate-walmart-apis`, `typescript-models`, `typescript-testing` |

## Agents

| Agent | What it does |
| --- | --- |
| `walmart-apis-planner` | Writes `walmart-apis-plan.md`, the plan and its contract sheet, before any project code is written, and revises it in place. `dotnet-integrate-walmart-apis`, `typescript-integrate-walmart-apis` and `python-integrate-walmart-apis` each start it once the SDK is installed or its source reached, on the main agent's own model. |

How each host gets it:

- **Claude Code** loads it from `scoped/agents/` with the plugin and lists it as `walmart-v4-apis:walmart-apis-planner`. Its file sets `model: inherit`, so it runs on the main agent's model.
- **Where no subagent tool lists it** (a host that doesn't load a plugin's agents, or subagents turned off), the main agent says so in one line and writes the plan itself before any code.

## Install

This plugin was **generated for you, not published to a shared plugin marketplace** — so you install it from a local path rather than by name. Point your coding agent at the directory holding this plugin, then install it:

```
/plugin marketplace add <path to the directory containing this plugin>
/plugin install walmart-v4-apis
```

In Codex the same two steps are CLI commands:

```
codex plugin marketplace add <path to the directory containing this plugin>
codex plugin add walmart-v4-apis@<marketplace>
```

`<marketplace>` is the name the directory you just added registers under — `codex plugin marketplace list` prints it.

Once your team publishes it to a plugin marketplace, add that marketplace in place of the local path and install `walmart-v4-apis` from it by name.

Then ask a usage question (e.g. *"how do I authenticate this SDK with an API key?"*) to trigger the relevant skill.

