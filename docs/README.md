# Agno Framework Documentation

This directory contains comprehensive documentation for the Agno framework, created based on the official documentation at **https://docs.agno.com**.

## Documents

### 0. [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) ⭐
Concise cheat sheet with code examples and common patterns:
- Core patterns (basic agent, tools, storage, RAG, memory, etc.)
- Execution methods comparison
- Model providers quick reference
- Database options
- Key parameters table
- Critical rules (what to avoid)
- When to use Agent vs Team vs Workflow
- Common tools list
- Production checklist
- Common errors and fixes

**Use this when:** You need quick code examples or want a cheat sheet reference.

### 1. [ARCHITECTURE.md](./ARCHITECTURE.md)
Complete architectural overview of the Agno framework including:
- Three-layer design (Build, Run, Manage)
- Five levels of agentic systems
- Core components (Agent, Team, Workflow)
- Data architecture and storage layers
- Execution models and flows
- Performance optimizations
- Security architecture
- Integration patterns
- Deployment strategies

**Use this when:** You want to understand how Agno works internally, its design principles, and overall system architecture.

### 2. [FEATURES_AND_CONFIGURATION.md](./FEATURES_AND_CONFIGURATION.md)
Detailed guide to all features and their configuration:
- Core features (model-agnostic, type-safe I/O, async-first, multimodal)
- Agent communication (Teams, A2A, Workflows)
- Persistent knowledge and RAG
- Memory system (user memory, culture/shared memory)
- Context loading and management
- Storage and persistence options
- Tools and integration (100+ built-in tools, custom tools, MCP)
- Guardrails and safety
- Human-in-the-loop
- Structured input/output
- Reasoning and advanced features
- Production features (AgentOS)
- **Cursor IDE configuration** (complete setup guide)

**Use this when:** You need to implement specific features, configure components, or set up your development environment.

### 3. [RESERVED_TERMS.md](./RESERVED_TERMS.md)
Complete glossary of all reserved terms and concepts:
- Core entities (Agent, Team, Workflow)
- Session management (session_id, run_id, user_id, agent_id)
- State and context (session_state, instructions, guidelines)
- Memory and knowledge (Memory, Knowledge, Culture)
- History and context loading
- Input/output schemas
- Tools and functions
- Database and storage
- Guardrails
- Human-in-the-loop
- Workflow-specific terms (Step, Condition, Loop, Parallel, Router)
- AgentOS terms
- Execution methods
- Response types
- Model configuration
- Debugging and monitoring
- Optimization features

**Use this when:** You need to understand specific terminology, look up configuration parameters, or reference framework concepts.

## Quick Start

### For New Users
1. Start with [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) for a quick overview
2. Read [ARCHITECTURE.md](./ARCHITECTURE.md) to understand the framework
3. Review the "Five Levels of Agentic Systems" section
4. Check execution flow diagrams

### For Developers
1. Use [QUICK_REFERENCE.md](./QUICK_REFERENCE.md) as your cheat sheet
2. Read [FEATURES_AND_CONFIGURATION.md](./FEATURES_AND_CONFIGURATION.md) for detailed configuration
3. Follow the Cursor IDE setup instructions
4. Reference [RESERVED_TERMS.md](./RESERVED_TERMS.md) as needed

### For Cursor Users
See the **Cursor IDE Configuration** section in [FEATURES_AND_CONFIGURATION.md](./FEATURES_AND_CONFIGURATION.md) for:
- Adding Agno documentation to Cursor
- Creating `.cursorrules` file
- Project structure recommendations
- Environment variable setup
- Quick start templates
- AI-assisted development tips

## Key Concepts Summary

### When to Use What

**Single Agent (90% of use cases):**
- One clear task or domain
- Can be solved with tools + instructions
- Example: Search, analyze, generate content

**Team (autonomous coordination):**
- Multiple specialized agents needed
- Agents decide who does what via LLM
- Complex tasks requiring multiple perspectives
- Example: Research + Analysis + Writing

**Workflow (programmatic control):**
- Sequential steps with clear flow
- Need conditional logic or branching
- Full control over execution order
- Example: ETL pipelines, multi-stage processing

### Critical Performance Rules

1. **NEVER create agents in loops** (529× slower)
2. **Use PostgreSQL in production** (not SQLite)
3. **Enable search_knowledge=True** for RAG
4. **Add history when context matters**
5. **Use output_schema for structured data**
6. **Start with single agent, scale up only when needed**

## Configuration in Cursor

### Quick Setup

1. **Add Agno docs to Cursor:**
   - Settings → Indexing & Docs
   - Add: `https://docs.agno.com/llms-full.txt`

2. **Create `.cursorrules` file:**
   ```
   You are an expert in Python, Agno framework, and AI agent development.
   
   Core Rules
   - NEVER create agents in loops - reuse them for performance
   - Always use output_schema for structured responses
   - PostgreSQL in production, SQLite for dev only
   - Start with single agent, scale up only when needed
   
   Docs: https://docs.agno.com
   ```

3. **Set environment variables:**
   ```bash
   export OPENAI_API_KEY=your-key
   export DATABASE_URL=postgresql://localhost/agno
   ```

### Works with Other IDEs
While these docs focus on Cursor, the framework works equally well with:
- VSCode with GitHub Copilot
- Windsurf
- Any Python IDE with AI assistance

## Production Checklist

- [ ] Using PostgreSQL (not SQLite)
- [ ] `debug_mode=False`
- [ ] `show_tool_calls=False`
- [ ] Agents created once and reused (never in loops)
- [ ] Error handling and retries configured
- [ ] Guardrails enabled for input validation
- [ ] Session isolation configured (user_id set)
- [ ] Knowledge base indexed and optimized
- [ ] Session summaries enabled
- [ ] Context compression configured
- [ ] Monitoring and tracing enabled
- [ ] RBAC and permissions configured

## Common Patterns

### Basic Agent
```python
from agno.agent import Agent
from agno.models.openai import OpenAIChat

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    instructions="You are a helpful assistant",
    markdown=True,
)
agent.print_response("Your query", stream=True)
```

### Agent with Storage
```python
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    add_history_to_context=True,
)
```

### Agent with Knowledge (RAG)
```python
from agno.knowledge.knowledge import Knowledge
from agno.vectordb.lancedb import LanceDb

knowledge = Knowledge(vector_db=LanceDb(uri="tmp/lancedb"))
knowledge.load_documents(paths=["docs/"])

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    knowledge=knowledge,
    search_knowledge=True,  # CRITICAL
)
```

### Agent with Memory
```python
from agno.memory import MemoryManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    memory=MemoryManager(),
    user_id="user-123",  # REQUIRED
)
```

### Multi-Agent Team
```python
from agno.team.team import Team

team = Team(
    members=[agent1, agent2, agent3],
    model=OpenAIChat(id="gpt-4o"),
    instructions="Coordinate team members",
)
```

### Sequential Workflow
```python
from agno.workflow.workflow import Workflow

async def pipeline(workflow, input_data):
    result1 = await agent1.arun(input_data)
    result2 = await agent2.arun(result1.content)
    return result2.content

workflow = Workflow(steps=pipeline)
```

## Additional Resources

- **Official Docs:** https://docs.agno.com
- **GitHub:** https://github.com/agno-agi/agno
- **Cookbook:** https://github.com/agno-agi/agno/tree/main/cookbook
- **Community:** https://community.agno.com
- **Discord:** https://discord.gg/4MtYHHrgA8

## Document Downloads

All documents in this directory are available as downloadable markdown files:
- `QUICK_REFERENCE.md` - Quick reference cheat sheet ⭐
- `ARCHITECTURE.md` - Framework architecture
- `FEATURES_AND_CONFIGURATION.md` - Features and configuration guide
- `RESERVED_TERMS.md` - Reserved terms glossary
- `README.md` - This file

Simply download or clone the repository to access all documentation offline.

**Recommended for printing:** `QUICK_REFERENCE.md` - fits on a few pages and covers most common patterns.

## Contributing

Found an error or want to improve the docs? These documents are based on the official Agno documentation. Please contribute to the main project at https://github.com/agno-agi/agno

## Version

These documents are current as of December 2025 and reflect Agno v2.2.2+.

For the most up-to-date information, always refer to the official documentation at https://docs.agno.com
