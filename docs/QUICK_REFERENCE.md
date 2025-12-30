# Agno Framework Quick Reference

A concise cheat sheet for the Agno framework. For detailed information, see the complete documentation files.

**Based on official documentation at https://docs.agno.com**

## Quick Links

- 📚 [Full Architecture](./ARCHITECTURE.md)
- ⚙️ [Features & Configuration](./FEATURES_AND_CONFIGURATION.md)
- 📖 [Reserved Terms Glossary](./RESERVED_TERMS.md)
- 🏠 [Documentation Index](./README.md)

---

## Core Patterns

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

### Agent + Tools
```python
from agno.tools.duckduckgo import DuckDuckGoTools

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[DuckDuckGoTools()],
    instructions="Search the web for information",
)
```

### Agent + Storage
```python
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    session_id="session-123",
    add_history_to_context=True,
    num_history_runs=5,
)
```

### Agent + Knowledge (RAG)
```python
from agno.knowledge.knowledge import Knowledge
from agno.vectordb.lancedb import LanceDb, SearchType
from agno.knowledge.embedder.openai import OpenAIEmbedder

knowledge = Knowledge(
    vector_db=LanceDb(
        uri="tmp/lancedb",
        table_name="docs",
        search_type=SearchType.hybrid,
        embedder=OpenAIEmbedder(id="text-embedding-3-small"),
    ),
)
knowledge.load_documents(paths=["docs/"])

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    knowledge=knowledge,
    search_knowledge=True,  # ⚠️ CRITICAL
)
```

### Agent + Memory
```python
from agno.memory import MemoryManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    memory=MemoryManager(),
    user_id="user-123",  # ⚠️ REQUIRED
    db=SqliteDb(db_file="agent.db"),
)
```

### Agent + State
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    session_state={"count": 0, "items": []},
)
```

### Structured Output
```python
from pydantic import BaseModel

class Response(BaseModel):
    summary: str
    confidence: float
    sources: list[str]

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    output_schema=Response,
)
result = agent.run("Analyze this")
response: Response = result.content  # Typed!
```

### Multi-Agent Team
```python
from agno.team.team import Team

researcher = Agent(name="Researcher", model=OpenAIChat(id="gpt-4o"), tools=[...])
writer = Agent(name="Writer", model=OpenAIChat(id="gpt-4o"))

team = Team(
    members=[researcher, writer],
    model=OpenAIChat(id="gpt-4o"),
    instructions="Research and write articles",
)
```

### Workflow
```python
from agno.workflow.workflow import Workflow

async def pipeline(workflow, topic: str):
    research = await researcher.arun(f"Research: {topic}")
    article = await writer.arun(f"Write about: {research.content}")
    return article.content

workflow = Workflow(name="Content Pipeline", steps=pipeline)
```

### Production (AgentOS)
```python
from agno.os import AgentOS
from agno.db.postgres import PostgresDb

agent_os = AgentOS(
    agents=[agent],
    db=PostgresDb(db_url="postgresql://localhost/prod"),
)
app = agent_os.get_app()  # FastAPI app

if __name__ == "__main__":
    agent_os.serve(app="main:app", port=7777)
```

---

## Execution Methods

| Method | Type | Returns | Use Case |
|--------|------|---------|----------|
| `run()` | Sync | `RunOutput` | Synchronous execution |
| `arun()` | Async | `RunOutput` | Async execution |
| `run_stream()` | Sync | `Iterator[Event]` | Sync streaming |
| `arun_stream()` | Async | `AsyncIterator[Event]` | Async streaming |
| `print_response()` | Sync | None | Quick testing |

**Examples:**
```python
# Sync
result = agent.run("Hello")
print(result.content)

# Async
result = await agent.arun("Hello")

# Stream
for chunk in agent.run_stream("Write essay"):
    print(chunk.content, end="", flush=True)

# Quick test
agent.print_response("Hello", stream=True)
```

---

## Model Providers

```python
# OpenAI
from agno.models.openai import OpenAIChat
model = OpenAIChat(id="gpt-4o")

# Anthropic
from agno.models.anthropic import Claude
model = Claude(id="claude-sonnet-4-5")

# Google
from agno.models.google import Gemini
model = Gemini(id="gemini-3-flash-preview")

# Local (Ollama)
from agno.models.ollama import Ollama
model = Ollama(id="llama3.1")
```

---

## Database Options

```python
# SQLite (dev only)
from agno.db.sqlite import SqliteDb
db = SqliteDb(db_file="agent.db")

# PostgreSQL (production - REQUIRED)
from agno.db.postgres import PostgresDb
db = PostgresDb(db_url="postgresql://localhost/prod")

# MySQL
from agno.db.mysql import MySQLDb
db = MySQLDb(db_url="mysql://localhost/ai")

# MongoDB
from agno.db.mongodb import MongoDb
db = MongoDb(connection_string="mongodb://localhost")
```

---

## Key Parameters

### Agent Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | Model | **Required** | LLM model to use |
| `instructions` | str | None | System message/behavior |
| `tools` | list | [] | Available tools |
| `knowledge` | Knowledge | None | Knowledge base |
| `memory` | MemoryManager | None | User memory |
| `db` | BaseDb | None | Database for persistence |
| `session_id` | str | Auto | Session identifier |
| `user_id` | str | None | User identifier |
| `session_state` | dict | {} | Session-level data |
| `input_schema` | BaseModel | None | Input validation |
| `output_schema` | BaseModel | None | Output structure |
| `guardrails` | list | [] | Input/output validation |
| `add_history_to_context` | bool | False | Include chat history |
| `num_history_runs` | int | None | Number of runs to include |
| `search_knowledge` | bool | False | Enable agentic RAG |
| `show_tool_calls` | bool | False | Show tool execution |
| `debug_mode` | bool | False | Verbose logging |
| `markdown` | bool | False | Format as markdown |

---

## Critical Rules

### ⚠️ NEVER Create Agents in Loops
```python
# ❌ WRONG - 529× slower
for query in queries:
    agent = Agent(...)
    agent.run(query)

# ✅ CORRECT
agent = Agent(...)
for query in queries:
    agent.run(query)
```

### ⚠️ PostgreSQL in Production
```python
# ❌ DEV ONLY
db = SqliteDb(db_file="agent.db")

# ✅ PRODUCTION
db = PostgresDb(db_url="postgresql://localhost/prod")
```

### ⚠️ Enable search_knowledge for RAG
```python
# ❌ Won't work
agent = Agent(knowledge=knowledge)

# ✅ Correct
agent = Agent(knowledge=knowledge, search_knowledge=True)
```

### ⚠️ Set user_id for Memory
```python
# ❌ Won't work
agent = Agent(memory=MemoryManager())

# ✅ Correct
agent = Agent(memory=MemoryManager(), user_id="user-123")
```

---

## When to Use What

### Single Agent (90% of cases)
- ✅ One clear task or domain
- ✅ Can be solved with tools + instructions
- ✅ Example: Search, analyze, generate

### Team (autonomous)
- ✅ Multiple specialized agents
- ✅ Complex tasks, different perspectives
- ✅ Example: Research + Analysis + Writing

### Workflow (programmatic)
- ✅ Sequential steps
- ✅ Conditional logic/branching
- ✅ Example: ETL, multi-stage processing

---

## Common Tools

```python
# Web Search
from agno.tools.duckduckgo import DuckDuckGoTools
from agno.tools.google_search import GoogleSearchTools

# Finance
from agno.tools.yfinance import YFinanceTools

# Email
from agno.tools.email import EmailTools

# Database
from agno.tools.sql import SQLTools

# Custom Function
def my_tool(arg: str) -> str:
    """Tool description for LLM."""
    return f"Result: {arg}"

agent = Agent(tools=[DuckDuckGoTools(), my_tool])
```

---

## Guardrails

```python
from agno.guardrails.pii import PIIGuardrail
from agno.guardrails.prompt_injection import PromptInjectionGuardrail

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    guardrails=[
        PIIGuardrail(),                    # Detect PII
        PromptInjectionGuardrail(),        # Detect injection
    ],
)
```

---

## Human-in-the-Loop

```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[EmailTools()],
    require_human_confirmation=True,  # Approval required
)
```

---

## Multimodal

```python
from agno.media import Image, Audio, Video, File

result = agent.run(
    "Analyze this",
    images=[Image.from_path("photo.jpg")],
    audios=[Audio.from_path("speech.mp3")],
    videos=[Video.from_path("video.mp4")],
    files=[File.from_path("doc.pdf")],
)
```

---

## Environment Variables

```bash
# LLM API Keys
export OPENAI_API_KEY=your-key
export ANTHROPIC_API_KEY=your-key
export GOOGLE_API_KEY=your-key

# Database
export DATABASE_URL=postgresql://localhost/agno

# Agno
export AGNO_TELEMETRY=false
```

---

## Cursor IDE Setup

> **📘 Complete Guide**: [CURSOR_SETUP_GUIDE.md](./CURSOR_SETUP_GUIDE.md)

### 1. Add Agno Docs (llms.txt protocol)
Settings → Indexing & Docs → Add:
```
https://docs.agno.com/llms-full.txt
```

### 2. Create Cursor Rules

**Simple projects** - `.cursorrules` file in project root:
```
You are an expert in Python, Agno framework, and AI agent development.

Core Rules
- NEVER create agents in loops - reuse them for performance
- Always use output_schema for structured responses
- PostgreSQL in production, SQLite for dev only
- Start with single agent, scale up only when needed

Docs: https://docs.agno.com
```

**Complex projects** - `.cursor/rules/` directory:
```
.cursor/
└── rules/
    ├── agno-general.mdc
    ├── agents.mdc
    └── production.mdc
```

### 3. Multi-Root Workspace (Optional)

For multiple services, create `.code-workspace`:
```json
{
  "folders": [
    { "path": ".", "name": "🏠 Root" },
    { "path": "services/research", "name": "🔍 Research" }
  ]
}
```

**See [CURSOR_SETUP_GUIDE.md](./CURSOR_SETUP_GUIDE.md) for complete configuration.**

---

## Production Checklist

- [ ] Using PostgreSQL (not SQLite)
- [ ] `debug_mode=False`
- [ ] `show_tool_calls=False`
- [ ] Agents created once and reused
- [ ] Error handling configured
- [ ] Guardrails enabled
- [ ] Session isolation (user_id)
- [ ] Knowledge indexed
- [ ] Summaries enabled
- [ ] Monitoring enabled

---

## Performance Stats

| Metric | Agno | LangGraph | Improvement |
|--------|------|-----------|-------------|
| Instantiation | 3μs | 1,587μs | **529× faster** |
| Memory | 6.6 KiB | 161 KiB | **24× less** |

---

## Common Errors & Fixes

### "Knowledge not being used"
```python
# ❌ Missing search_knowledge
agent = Agent(knowledge=knowledge)

# ✅ Enable search
agent = Agent(knowledge=knowledge, search_knowledge=True)
```

### "Memory not persisting"
```python
# ❌ Missing user_id or db
agent = Agent(memory=MemoryManager())

# ✅ Add user_id and db
agent = Agent(
    memory=MemoryManager(),
    user_id="user-123",
    db=SqliteDb(db_file="agent.db"),
)
```

### "Slow performance"
```python
# ❌ Creating in loop
for q in queries:
    agent = Agent(...)  # Don't do this!

# ✅ Create once
agent = Agent(...)
for q in queries:
    agent.run(q)
```

---

## Resources

- 📚 Docs: https://docs.agno.com
- 💻 GitHub: https://github.com/agno-agi/agno
- 🍳 Cookbook: https://github.com/agno-agi/agno/tree/main/cookbook
- 💬 Community: https://community.agno.com
- 🎮 Discord: https://discord.gg/4MtYHHrgA8
- 🎨 UI: https://os.agno.com

---

## Version

Current: Agno v2.2.2+ (December 2025)

For detailed information, see the complete documentation files in this directory.
