# Complete Cursor IDE Setup Guide for Agno Framework

This guide provides comprehensive Cursor-specific configuration for working with the Agno framework, using proper Cursor terminology and concepts.

**Based on official Cursor documentation and https://docs.agno.com**

---

## Table of Contents

1. [Understanding Cursor Concepts](#understanding-cursor-concepts)
2. [Documentation Indexing](#documentation-indexing)
3. [Cursor Rules Configuration](#cursor-rules-configuration)
4. [Workspace Setup](#workspace-setup)
5. [Project Structure](#project-structure)
6. [Multi-Root Workspace Configuration](#multi-root-workspace-configuration)
7. [Best Practices](#best-practices)

---

## Understanding Cursor Concepts

### Cursor Rules
Cursor Rules are custom instructions that persistently guide Cursor's AI for code analysis, formatting, and generation. They help enforce project-specific standards and conventions.

**Two types of rules:**
- **Global Rules**: Apply across all workspaces (managed in Cursor settings)
- **Project Rules**: Specific to your workspace (stored in `.cursor/rules/` or `.cursorrules`)

### llms.txt Protocol
The `llms.txt` protocol structures documentation for AI consumption. Cursor can index these files to provide better context-aware suggestions.

- `llms.txt` - Basic documentation index with links
- `llms-full.txt` - Complete marked-up documentation (recommended for Cursor)

### Workspace Files
`.code-workspace` files configure multi-root workspaces in Cursor, allowing you to work with multiple folders simultaneously.

---

## Documentation Indexing

### Step 1: Add Agno Documentation to Cursor

1. **Open Cursor Settings**
   - Mac: `Cmd + ,`
   - Windows/Linux: `Ctrl + ,`

2. **Navigate to Documentation**
   - Go to **Settings → Indexing & Docs** (or search for "docs" in settings)

3. **Add Agno Documentation**
   - Click **"Add New Doc"**
   - Enter: `https://docs.agno.com/llms-full.txt`
   - Click **"Add"**

4. **Wait for Indexing**
   - Cursor will download and index the documentation
   - This provides context-aware code completion and answers

### What This Gives You

With indexed documentation, Cursor can:
- Answer questions about Agno framework directly
- Suggest Agno-specific code patterns
- Provide accurate parameter names and types
- Reference official examples in completions

---

## Cursor Rules Configuration

Cursor Rules ensure consistent AI behavior across your Agno project. You can use either a single `.cursorrules` file or the `.cursor/rules/` directory structure.

### Option 1: Single .cursorrules File (Recommended for Simple Projects)

Create `.cursorrules` in your **project root**:

```
# Agno Framework - Cursor AI Rules
# Version: 1.0

You are an expert in Python, Agno framework, and AI agent development.

## Core Performance Rules (CRITICAL)

1. NEVER create agents in loops - create once and reuse
   - Agent instantiation: 3μs vs 1,587μs (529× slower if recreated)
   - Memory: 6.6 KiB vs 161 KiB (24× higher if recreated)

2. NEVER forget search_knowledge=True when using Knowledge
   - Knowledge base won't be used without this flag

3. NEVER use user_id without Memory
   - Memory requires both user_id and db parameters

## Database Rules

- Development: SqliteDb (local only)
- Production: PostgresDb (REQUIRED)
- Never use SQLite in production

## Agent Patterns

### Basic Agent (90% of use cases)
Use a single agent when:
- One clear task or domain
- Can be solved with tools + instructions
- No need for multiple perspectives

```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    instructions="You are a helpful assistant",
    markdown=True,
)
```

### Team (Autonomous Collaboration)
Use Team when:
- Multiple specialized agents needed
- Agents decide who does what via LLM
- Complex tasks requiring different expertise

```python
from agno.team.team import Team

team = Team(
    members=[agent1, agent2],
    model=OpenAIChat(id="gpt-4o"),
    instructions="Coordinate team members",
)
```

### Workflow (Programmatic Control)
Use Workflow when:
- Sequential steps with clear flow
- Conditional logic or branching needed
- Full control over execution order

```python
from agno.workflow.workflow import Workflow

async def pipeline(workflow, data):
    result1 = await agent1.arun(data)
    result2 = await agent2.arun(result1.content)
    return result2.content

workflow = Workflow(steps=pipeline)
```

## Knowledge Base (RAG) Rules

ALWAYS set search_knowledge=True:
```python
agent = Agent(
    knowledge=knowledge,
    search_knowledge=True,  # CRITICAL
)
```

## Memory Rules

ALWAYS set user_id with Memory:
```python
agent = Agent(
    memory=MemoryManager(),
    user_id="user-123",  # REQUIRED
    db=SqliteDb(db_file="agent.db"),
)
```

## Context Management

Add history for conversational agents:
```python
agent = Agent(
    add_history_to_context=True,
    num_history_runs=5,
)
```

## Production Checklist

Before deployment:
- [ ] PostgreSQL database (not SQLite)
- [ ] debug_mode=False
- [ ] show_tool_calls=False
- [ ] Agents created once and reused
- [ ] Guardrails configured
- [ ] Session isolation (user_id set)
- [ ] Error handling implemented

## Common Mistakes to Avoid

1. Creating agents in loops
2. Forgetting search_knowledge=True
3. Using Team when single agent works
4. Using SQLite in production
5. Not adding history when context matters
6. Missing output_schema for structured data

## Documentation

Official docs: https://docs.agno.com
Cookbook: https://github.com/agno-agi/agno/tree/main/cookbook
```

### Option 2: Structured .cursor/rules/ Directory (Recommended for Complex Projects)

For larger projects, organize rules by file type or domain:

```
.cursor/
└── rules/
    ├── agno-general.mdc       # General Agno rules
    ├── agents.mdc             # Agent-specific rules
    ├── teams.mdc              # Team-specific rules
    ├── workflows.mdc          # Workflow-specific rules
    └── production.mdc         # Production deployment rules
```

#### Example: `.cursor/rules/agno-general.mdc`

```yaml
---
files: "**/*.py"
---

You are an expert in Python and the Agno framework.

# Performance Rules
- NEVER create agents in loops - reuse them (529× faster)
- Always use PostgreSQL in production, never SQLite

# Documentation
Reference: https://docs.agno.com
```

#### Example: `.cursor/rules/agents.mdc`

```yaml
---
files: "**/agents/**/*.py"
---

You are configuring Agno agents.

# Agent Configuration Rules

1. Always specify model explicitly
2. Use output_schema for structured responses
3. Set search_knowledge=True when using Knowledge
4. Set user_id when using Memory
5. Add history for conversational agents

# Templates

Basic agent:
```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat

agent = Agent(
    name="Agent Name",
    model=OpenAIChat(id="gpt-4o"),
    instructions="Clear instructions here",
    tools=[],  # Add tools as needed
    markdown=True,
)
```

Agent with storage:
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    add_history_to_context=True,
)
```
```

#### Example: `.cursor/rules/production.mdc`

```yaml
---
files: "**/production/**/*.py, **/deploy/**/*.py"
---

You are configuring production Agno deployments.

# Production Requirements

ALWAYS use:
- PostgreSQL database (PostgresDb)
- debug_mode=False
- show_tool_calls=False
- Error handling and retries
- Guardrails for input validation
- Proper logging

NEVER use:
- SQLite database
- Debug mode enabled
- Hardcoded credentials

# AgentOS Production Template

```python
from agno.os import AgentOS
from agno.db.postgres import PostgresDb
import os

agent_os = AgentOS(
    agents=[agent],
    db=PostgresDb(db_url=os.getenv("DATABASE_URL")),
)

app = agent_os.get_app()

if __name__ == "__main__":
    agent_os.serve(
        app="main:app",
        host="0.0.0.0",
        port=7777,
    )
```
```

### Creating Rules via Cursor Command Palette

1. Open Command Palette: `Cmd/Ctrl + Shift + P`
2. Type: **"New Cursor Rule"**
3. Choose location:
   - Project rule: `.cursor/rules/`
   - Global rule: Cursor settings
4. Name your rule file
5. Add your instructions

---

## Workspace Setup

### Single-Root Workspace (Simple Projects)

For a single Agno project, no workspace file is needed. Just:

1. Open your project folder in Cursor
2. Add `.cursorrules` file
3. Configure environment variables in `.env`

**Project Structure:**
```
my-agno-project/
├── .cursorrules           # Cursor AI rules
├── .env                   # Environment variables
├── .gitignore
├── requirements.txt
├── agents/                # Agent definitions
├── teams/                 # Team configurations
├── workflows/             # Workflow definitions
└── main.py               # Entry point
```

### Multi-Root Workspace (Complex Projects)

For monorepos or multi-service projects, use a `.code-workspace` file.

---

## Multi-Root Workspace Configuration

### When to Use Multi-Root Workspaces

Use multi-root when you have:
- Multiple related Agno services
- Separate frontend/backend/agents repositories
- Shared libraries across projects
- Documentation as a separate folder

### Creating a Multi-Root Workspace

#### Method 1: Via Cursor UI

1. Open first folder in Cursor
2. Go to **File → Add Folder to Workspace...**
3. Select additional folders
4. Save via **File → Save Workspace As...**
5. Name it `agno-workspace.code-workspace`

#### Method 2: Manual Creation

Create `agno-workspace.code-workspace`:

```json
{
  "folders": [
    {
      "path": ".",
      "name": "🏠 Root"
    },
    {
      "path": "services/research-agent",
      "name": "🔍 Research Agent"
    },
    {
      "path": "services/writer-agent",
      "name": "✍️ Writer Agent"
    },
    {
      "path": "services/analyst-agent",
      "name": "📊 Analyst Agent"
    },
    {
      "path": "shared/agno-tools",
      "name": "🛠️ Shared Tools"
    },
    {
      "path": "docs",
      "name": "📚 Documentation"
    }
  ],
  "settings": {
    // Python settings
    "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.formatting.provider": "black",
    
    // Editor settings
    "editor.formatOnSave": true,
    "editor.rulers": [88],
    "editor.tabSize": 4,
    
    // Agno-specific settings
    "files.exclude": {
      "**/__pycache__": true,
      "**/*.pyc": true,
      "**/tmp": true,
      "**/.pytest_cache": true,
      "**/agent.db": false,  // Show SQLite files
      "**/lancedb": false     // Show vector DB
    },
    
    // File associations
    "files.associations": {
      ".cursorrules": "plaintext",
      "*.mdc": "markdown"
    }
  },
  "extensions": {
    "recommendations": [
      "ms-python.python",
      "ms-python.vscode-pylance",
      "charliermarsh.ruff",
      "ms-toolsai.jupyter",
      "tamasfe.even-better-toml"
    ]
  },
  "launch": {
    "version": "0.2.0",
    "configurations": [
      {
        "name": "AgentOS: Research Service",
        "type": "python",
        "request": "launch",
        "program": "${workspaceFolder}/services/research-agent/main.py",
        "console": "integratedTerminal",
        "env": {
          "PYTHONPATH": "${workspaceFolder}"
        }
      },
      {
        "name": "AgentOS: All Services",
        "type": "python",
        "request": "launch",
        "program": "${workspaceFolder}/main.py",
        "console": "integratedTerminal"
      }
    ]
  },
  "tasks": {
    "version": "2.0.0",
    "tasks": [
      {
        "label": "Install Dependencies",
        "type": "shell",
        "command": "pip install -r requirements.txt",
        "group": "build"
      },
      {
        "label": "Run Tests",
        "type": "shell",
        "command": "pytest",
        "group": "test"
      },
      {
        "label": "Load Knowledge Base",
        "type": "shell",
        "command": "python scripts/load_knowledge.py"
      }
    ]
  }
}
```

### Multi-Root Best Practices

1. **Naming Convention**: Use emojis or prefixes for clarity
2. **Relative Paths**: Use relative paths from workspace file location
3. **Shared Settings**: Put common settings in workspace file
4. **Per-Folder Rules**: Add `.cursor/rules/` in each folder for specific behavior
5. **Context Hints**: Add README in root explaining folder relationships

### Working with Multi-Root in Cursor

**Referencing Files Across Roots:**
```python
# In Cursor chat, use @ to reference files
# @research-agent/agents/researcher.py
# @shared-tools/agno_helpers.py
```

**Folder-Specific Rules:**
Each root can have its own `.cursor/rules/`:
```
services/research-agent/.cursor/rules/research.mdc
services/writer-agent/.cursor/rules/writing.mdc
shared/agno-tools/.cursor/rules/tools.mdc
```

---

## Project Structure

### Recommended Structure for Agno Projects

```
agno-project/
├── .cursorrules                    # Cursor AI rules (or use .cursor/)
├── .cursor/
│   └── rules/
│       ├── agno-general.mdc
│       ├── agents.mdc
│       └── production.mdc
├── .code-workspace                 # Optional: for multi-root
├── .env                            # Environment variables
├── .gitignore
├── requirements.txt
│
├── agents/                         # Agent definitions
│   ├── __init__.py
│   ├── research_agent.py
│   ├── writer_agent.py
│   └── analyst_agent.py
│
├── teams/                          # Team configurations
│   ├── __init__.py
│   └── content_team.py
│
├── workflows/                      # Workflow definitions
│   ├── __init__.py
│   └── content_pipeline.py
│
├── tools/                          # Custom tools
│   ├── __init__.py
│   └── custom_tools.py
│
├── knowledge/                      # Knowledge base files
│   ├── documents/
│   └── load_knowledge.py
│
├── data/                           # Data storage
│   ├── agent.db                    # SQLite (dev)
│   └── lancedb/                    # Vector DB
│
├── config/                         # Configuration files
│   ├── __init__.py
│   └── settings.py
│
├── tests/                          # Tests
│   ├── __init__.py
│   ├── test_agents.py
│   └── test_workflows.py
│
├── scripts/                        # Utility scripts
│   └── setup_db.py
│
└── main.py                         # AgentOS entry point
```

---

## Best Practices

### 1. Rule Organization

**Small Projects:**
- Use single `.cursorrules` file
- Keep it under 200 lines
- Focus on most critical rules

**Large Projects:**
- Use `.cursor/rules/` directory
- One rule file per domain (agents, teams, workflows)
- Use YAML frontmatter for file patterns

### 2. Documentation References

In your rules, reference specific Agno docs:
```
For agent configuration, see: https://docs.agno.com/agents/configuration
For team setup, see: https://docs.agno.com/teams/overview
```

### 3. Context Management

Help Cursor understand your project:
```
# In .cursorrules or rule files

## Project Context
This is an Agno multi-agent system with:
- Research agent: Web search and data gathering
- Writer agent: Content generation
- Analyst agent: Data analysis

Agents communicate via a Team with autonomous coordination.
Database: PostgreSQL in production, SQLite in dev.
```

### 4. File Pattern Specificity

Use specific patterns in `.cursor/rules/*.mdc`:
```yaml
---
files: "**/agents/**/*.py"
---
# Rules apply only to agent files
```

### 5. Version Control

**Commit to Git:**
- `.cursorrules` or `.cursor/rules/`
- `.code-workspace` (for multi-root)
- `.gitignore` should exclude:
  - `*.db` (SQLite databases)
  - `lancedb/` (vector databases)
  - `.env` (secrets)
  - `__pycache__/`

**Example .gitignore:**
```gitignore
# Agno-specific
*.db
lancedb/
tmp/

# Python
__pycache__/
*.py[cod]
.venv/
.pytest_cache/

# Environment
.env
.env.local

# Cursor (keep rules, ignore cache)
.cursor/*
!.cursor/rules/
```

### 6. Team Collaboration

Share with your team:
1. Commit `.cursorrules` or `.cursor/rules/` to Git
2. Document in README how to set up Cursor
3. Include workspace file for multi-root projects
4. Keep rules updated as patterns evolve

### 7. Testing Rules

After creating/updating rules:
1. Open a Python file
2. Ask Cursor to "create an Agno agent"
3. Verify it follows your rules
4. Adjust rules if needed

---

## Quick Start Checklist

- [ ] Add `https://docs.agno.com/llms-full.txt` to Cursor docs
- [ ] Create `.cursorrules` or `.cursor/rules/` directory
- [ ] Add performance rules (agent reuse, database choice)
- [ ] Configure `.code-workspace` (if multi-root)
- [ ] Set up environment variables in `.env`
- [ ] Add `.gitignore` for Agno-specific files
- [ ] Test rules by asking Cursor to generate code
- [ ] Commit rules to version control

---

## Troubleshooting

### Cursor Not Using Rules

1. **Check file location**: `.cursorrules` must be in project root
2. **Check syntax**: Plain text, no special formatting needed
3. **Restart Cursor**: After adding rules
4. **Check file patterns**: In `.mdc` files, verify `files:` pattern

### Documentation Not Indexed

1. **Verify URL**: `https://docs.agno.com/llms-full.txt` is correct
2. **Check network**: Cursor needs internet to download docs
3. **Wait for completion**: Indexing can take a few minutes
4. **Check settings**: Settings → Indexing & Docs

### Multi-Root Context Issues

1. **Use @ references**: `@folder-name/file.py` in Cursor chat
2. **Specify folder**: Be explicit about which root you're working in
3. **Check workspace file**: Verify folder paths are correct
4. **Reload window**: Cmd/Ctrl + Shift + P → "Reload Window"

---

## Advanced Configuration

### Custom Keyboard Shortcuts

Add to Cursor keybindings (JSON):
```json
[
  {
    "key": "cmd+shift+a",
    "command": "workbench.action.terminal.sendSequence",
    "args": { "text": "python main.py\n" },
    "when": "terminalFocus"
  },
  {
    "key": "cmd+shift+t",
    "command": "workbench.action.terminal.sendSequence",
    "args": { "text": "pytest\n" },
    "when": "terminalFocus"
  }
]
```

### Snippets for Agno

Add to Python snippets (`.cursor/snippets/python.json`):
```json
{
  "Agno Basic Agent": {
    "prefix": "agno-agent",
    "body": [
      "from agno.agent import Agent",
      "from agno.models.openai import OpenAIChat",
      "",
      "agent = Agent(",
      "    name=\"${1:Agent Name}\",",
      "    model=OpenAIChat(id=\"gpt-4o\"),",
      "    instructions=\"${2:Instructions}\",",
      "    markdown=True,",
      ")",
      "$0"
    ],
    "description": "Create a basic Agno agent"
  }
}
```

---

## Resources

- **Cursor Documentation**: https://cursor.sh/docs
- **Agno Documentation**: https://docs.agno.com
- **Agno Cookbook**: https://github.com/agno-agi/agno/tree/main/cookbook
- **Cursor Rules Guide**: https://learn-cursor.com/en/rules/
- **Multi-Root Workspaces**: https://code.visualstudio.com/docs/editing/workspaces

---

## Summary

This guide covered:
- ✅ Cursor-specific terminology (rules, workspace, llms.txt)
- ✅ Documentation indexing in Cursor
- ✅ Two approaches to Cursor Rules (.cursorrules vs .cursor/rules/)
- ✅ Multi-root workspace configuration with .code-workspace
- ✅ Project structure recommendations
- ✅ Best practices for team collaboration
- ✅ Troubleshooting common issues

For more details on Agno features, see [FEATURES_AND_CONFIGURATION.md](./FEATURES_AND_CONFIGURATION.md)
