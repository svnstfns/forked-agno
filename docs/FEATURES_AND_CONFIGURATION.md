# Agno Framework Features and Configuration Guide

This guide covers all major features of the Agno framework and how to configure them properly, with special attention to Cursor IDE setup.

**Based on official documentation at https://docs.agno.com**

## Table of Contents
1. [Core Features](#core-features)
2. [Agent Communication](#agent-communication)
3. [Persistent Knowledge](#persistent-knowledge)
4. [Memory System](#memory-system)
5. [Context Loading and Management](#context-loading-and-management)
6. [Storage and Persistence](#storage-and-persistence)
7. [Tools and Integration](#tools-and-integration)
8. [Guardrails and Safety](#guardrails-and-safety)
9. [Human-in-the-Loop](#human-in-the-loop)
10. [Structured Input/Output](#structured-inputoutput)
11. [Reasoning and Advanced Features](#reasoning-and-advanced-features)
12. [Production Features (AgentOS)](#production-features-agentos)
13. [Cursor IDE Configuration](#cursor-ide-configuration)

---

## Core Features

### 1. Model Agnostic
Works with any LLM provider with a consistent interface.

**Configuration:**
```python
from agno.agent import Agent

# OpenAI
from agno.models.openai import OpenAIChat
agent = Agent(model=OpenAIChat(id="gpt-4o"))

# Anthropic
from agno.models.anthropic import Claude
agent = Agent(model=Claude(id="claude-sonnet-4-5"))

# Google
from agno.models.google import Gemini
agent = Agent(model=Gemini(id="gemini-3-flash-preview"))

# Local (Ollama)
from agno.models.ollama import Ollama
agent = Agent(model=Ollama(id="llama3.1"))
```

**Key Parameters:**
- `id`: Model identifier
- `temperature`: Controls randomness (0.0-1.0)
- `max_tokens`: Maximum response length
- `api_key`: API key (can also use environment variables)

### 2. Type-Safe I/O
Validate inputs and outputs using Pydantic schemas.

**Configuration:**
```python
from pydantic import BaseModel
from agno.agent import Agent

class UserQuery(BaseModel):
    question: str
    context: str = None

class AgentResponse(BaseModel):
    answer: str
    confidence: float
    sources: list[str]

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    input_schema=UserQuery,      # Validates input
    output_schema=AgentResponse,  # Structures output
)

# Usage
result = agent.run(UserQuery(question="What is Agno?"))
response: AgentResponse = result.content  # Typed response
```

### 3. Async-First Design
Built for long-running tasks and concurrent operations.

**Configuration:**
```python
import asyncio
from agno.agent import Agent

agent = Agent(model=OpenAIChat(id="gpt-4o"))

# Async execution
async def main():
    result = await agent.arun("Process this asynchronously")
    
    # Stream async
    async for chunk in agent.arun_stream("Stream this"):
        print(chunk.content, end="", flush=True)

asyncio.run(main())
```

### 4. Multimodal Support
Native support for text, images, audio, video, and files.

**Configuration:**
```python
from agno.agent import Agent
from agno.media import Image, Audio, Video, File

agent = Agent(model=OpenAIChat(id="gpt-4o"))

# Image analysis
result = agent.run(
    "What's in this image?",
    images=[Image.from_path("photo.jpg")]
)

# Audio processing
result = agent.run(
    "Transcribe this",
    audios=[Audio.from_path("speech.mp3")]
)

# Multiple modalities
result = agent.run(
    "Analyze this presentation",
    images=[Image.from_path("slide1.jpg"), Image.from_path("slide2.jpg")],
    files=[File.from_path("notes.pdf")]
)
```

---

## Agent Communication

### 1. Team-Based Communication
Agents collaborate autonomously via Teams.

**Configuration:**
```python
from agno.agent import Agent
from agno.team.team import Team
from agno.tools.duckduckgo import DuckDuckGoTools

# Create specialized agents
researcher = Agent(
    name="Researcher",
    model=OpenAIChat(id="gpt-4o"),
    tools=[DuckDuckGoTools()],
    instructions="Research topics thoroughly using web search"
)

writer = Agent(
    name="Writer",
    model=OpenAIChat(id="gpt-4o"),
    instructions="Write clear, engaging content"
)

analyst = Agent(
    name="Analyst",
    model=OpenAIChat(id="gpt-4o"),
    instructions="Analyze data and provide insights"
)

# Create team
team = Team(
    name="Content Team",
    members=[researcher, writer, analyst],
    model=OpenAIChat(id="gpt-4o"),  # Leader LLM
    instructions="Coordinate team to research, analyze, and write content",
)

# Team decides which agent(s) to use
team.print_response("Research AI trends and write an analysis", stream=True)
```

**Key Features:**
- Autonomous delegation by leader LLM
- Shared team context and memory
- Dynamic agent selection based on task

### 2. Agent-to-Agent (A2A) Protocol
Direct agent communication for complex workflows.

**Configuration:**
```python
from agno.agent import Agent
from agno.tools.a2a import A2ATools

# Agent 1 can call Agent 2
agent2 = Agent(
    name="Specialist",
    model=OpenAIChat(id="gpt-4o"),
    instructions="Provide specialized analysis"
)

agent1 = Agent(
    name="Coordinator",
    model=OpenAIChat(id="gpt-4o"),
    tools=[A2ATools(agents=[agent2])],
    instructions="Coordinate with specialists when needed"
)
```

### 3. Workflow-Based Communication
Programmatic agent orchestration.

**Configuration:**
```python
from agno.agent import Agent
from agno.workflow.workflow import Workflow

researcher = Agent(name="Researcher", model=OpenAIChat(id="gpt-4o"))
writer = Agent(name="Writer", model=OpenAIChat(id="gpt-4o"))

async def content_pipeline(workflow, topic: str):
    # Step 1: Research
    research = await researcher.arun(f"Research: {topic}")
    
    # Step 2: Write
    article = await writer.arun(
        f"Write article about: {topic}\nResearch: {research.content}"
    )
    
    return article.content

workflow = Workflow(
    name="Content Pipeline",
    steps=content_pipeline,
)

workflow.print_response("AI in Healthcare", stream=True)
```

---

## Persistent Knowledge

### Configuration and Usage

**Basic Setup:**
```python
from agno.agent import Agent
from agno.knowledge.knowledge import Knowledge
from agno.vectordb.lancedb import LanceDb, SearchType
from agno.knowledge.embedder.openai import OpenAIEmbedder

# Create knowledge base
knowledge = Knowledge(
    vector_db=LanceDb(
        uri="tmp/lancedb",
        table_name="knowledge_base",
        search_type=SearchType.hybrid,  # Vector + keyword search
        embedder=OpenAIEmbedder(id="text-embedding-3-small"),
    ),
)

# Load documents
knowledge.load_documents(
    paths=["docs/", "data/"],
    include=["*.pdf", "*.md", "*.txt"],
    exclude=["*draft*"],
)

# Create agent with knowledge
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    knowledge=knowledge,
    search_knowledge=True,  # CRITICAL: enables agentic RAG
    instructions="Use knowledge base and cite sources",
)
```

**Supported Vector Databases:**
- LanceDB (local, serverless)
- Pinecone (cloud)
- Qdrant (local or cloud)
- Weaviate (local or cloud)
- PostgreSQL with pgvector
- ChromaDB
- Milvus

**Configuration Options:**
```python
from agno.vectordb.pgvector import PgVector

knowledge = Knowledge(
    vector_db=PgVector(
        table_name="documents",
        db_url="postgresql://localhost/ai",
        embedder=OpenAIEmbedder(id="text-embedding-3-small"),
    ),
    max_results=10,  # Number of results to retrieve
)
```

**Advanced Features:**
- **Hybrid Search**: Combines vector similarity and keyword matching
- **Reranking**: Re-scores results for better relevance
- **Filtering**: Filter by metadata (date, author, tags)
- **Chunking Strategies**: Custom document splitting

---

## Memory System

Memory allows agents to remember user preferences and facts across sessions.

### User Memory Configuration

**Basic Setup:**
```python
from agno.agent import Agent
from agno.memory import MemoryManager
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    user_id="user-123",  # REQUIRED for memory
    memory=MemoryManager(),  # Auto-captures memories
    add_history_to_context=True,
)
```

**Custom Memory Manager:**
```python
from agno.memory import MemoryManager

memory_manager = MemoryManager(
    model=OpenAIChat(id="gpt-4o"),
    add_memories=True,      # Capture new memories
    update_memories=True,   # Update existing memories
    delete_memories=False,  # Don't delete memories
    system_message="Custom memory instructions...",
)

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    memory=memory_manager,
    user_id="user-123",
)
```

**Memory Storage:**
- Stored in database (user_memories table)
- Persists across sessions
- Associated with user_id
- Automatically retrieved on each run

### Culture (Shared Memory)

Culture is shared memory across all agents in a system.

**Configuration:**
```python
from agno.culture.manager import CultureManager

culture = CultureManager(
    db=SqliteDb(db_file="shared.db"),
    model=OpenAIChat(id="gpt-4o"),
)

# Multiple agents share culture
agent1 = Agent(culture=culture, ...)
agent2 = Agent(culture=culture, ...)
agent3 = Agent(culture=culture, ...)
```

**Use Cases:**
- Organization-wide knowledge
- Team conventions and practices
- Shared context across agents

---

## Context Loading and Management

### History Management

**Configuration:**
```python
from agno.agent import Agent

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    session_id="session-123",
    add_history_to_context=True,  # Load previous messages
    num_history_runs=5,            # Last 5 conversations
)
```

**Options:**
- `add_history_to_context`: Include previous messages (default: False)
- `num_history_runs`: Number of previous runs to include
- `add_datetime_to_context`: Add timestamp to messages

### Session Summaries

For long conversations, use summaries to reduce context size.

**Configuration:**
```python
from agno.session.summary import SessionSummaryManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    session_summary_manager=SessionSummaryManager(
        num_runs_to_summarize=10,  # Summarize every 10 runs
    ),
)
```

### Context Compression

**Configuration:**
```python
from agno.compression.manager import CompressionManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    compression_manager=CompressionManager(
        compress_messages=True,
        compression_ratio=0.5,  # Target 50% reduction
    ),
)
```

---

## Storage and Persistence

### Session Storage

**SQLite (Development):**
```python
from agno.db.sqlite import SqliteDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="tmp/agent.db"),
    session_id="session-123",
    user_id="user-456",
)
```

**PostgreSQL (Production - REQUIRED):**
```python
from agno.db.postgres import PostgresDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=PostgresDb(db_url="postgresql://localhost/prod"),
    session_id="session-123",
    user_id="user-456",
)
```

**Other Databases:**
```python
# MySQL
from agno.db.mysql import MySQLDb
db = MySQLDb(db_url="mysql://localhost/ai")

# MongoDB
from agno.db.mongodb import MongoDb
db = MongoDb(connection_string="mongodb://localhost")

# DynamoDB
from agno.db.dynamodb import DynamoDb
db = DynamoDb(table_name="agents", region="us-east-1")

# Redis
from agno.db.redis import RedisDb
db = RedisDb(redis_url="redis://localhost:6379")
```

### Session State

Track structured data across runs.

**Configuration:**
```python
from agno.agent import Agent

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=SqliteDb(db_file="agent.db"),
    session_state={"shopping_list": [], "budget": 1000},
)

# State persists across runs
result1 = agent.run("Add milk to my list")
result2 = agent.run("What's on my list?")

# Access state programmatically
session = agent.get_session()
print(session.session_state["shopping_list"])
```

---

## Tools and Integration

### Built-in Tools

**Available Toolkits (100+):**
- Web: DuckDuckGo, Google Search, Brave Search
- Finance: YFinance, Alpha Vantage
- Database: SQL, MongoDB, Redis
- Cloud: AWS, GCP, Azure
- Communication: Email, Slack, Discord
- Data: CSV, Excel, PDF processing
- And many more...

**Configuration:**
```python
from agno.agent import Agent
from agno.tools.duckduckgo import DuckDuckGoTools
from agno.tools.yfinance import YFinanceTools
from agno.tools.email import EmailTools

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[
        DuckDuckGoTools(),
        YFinanceTools(),
        EmailTools(smtp_host="smtp.gmail.com"),
    ],
    show_tool_calls=True,  # Show tool execution
)
```

### Custom Tools

**Function-based:**
```python
from agno.agent import Agent

def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"Weather in {city}: Sunny, 72°F"

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[get_weather],  # Automatically converted to tool
)
```

**Toolkit-based:**
```python
from agno.tools import Toolkit
from agno.tools.function import Function

class MyToolkit(Toolkit):
    def __init__(self):
        super().__init__(name="my_toolkit")
        self.register(Function(name="tool1", fn=self.tool1))
        self.register(Function(name="tool2", fn=self.tool2))
    
    def tool1(self, arg: str) -> str:
        """Tool 1 description."""
        return f"Result: {arg}"
    
    def tool2(self, arg: int) -> int:
        """Tool 2 description."""
        return arg * 2

agent = Agent(tools=[MyToolkit()])
```

### MCP (Model Context Protocol) Tools

**Configuration:**
```python
from agno.tools.mcp import MCPTools

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[
        MCPTools(
            transport="streamable-http",
            url="https://docs.agno.com/mcp"
        )
    ],
)
```

---

## Guardrails and Safety

### Input/Output Validation

**Configuration:**
```python
from agno.agent import Agent
from agno.guardrails.pii import PIIGuardrail
from agno.guardrails.prompt_injection import PromptInjectionGuardrail

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    guardrails=[
        PIIGuardrail(),                    # Detect PII
        PromptInjectionGuardrail(),        # Detect prompt injection
    ],
)
```

**Custom Guardrails:**
```python
from agno.guardrails.base import BaseGuardrail
from agno.exceptions import GuardrailError

class ContentPolicyGuardrail(BaseGuardrail):
    def validate_input(self, input_text: str) -> str:
        if "banned_word" in input_text.lower():
            raise GuardrailError("Content policy violation")
        return input_text
    
    def validate_output(self, output_text: str) -> str:
        if "sensitive_info" in output_text.lower():
            return output_text.replace("sensitive_info", "[REDACTED]")
        return output_text

agent = Agent(guardrails=[ContentPolicyGuardrail()])
```

---

## Human-in-the-Loop

Require human approval before executing tools.

**Configuration:**
```python
from agno.agent import Agent
from agno.tools.email import EmailTools

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[EmailTools()],
    require_human_confirmation=True,  # Require approval for ALL tools
)

# Or per-tool confirmation
def sensitive_action(data: str) -> str:
    """Sensitive action requiring approval."""
    return f"Processed: {data}"

sensitive_action._require_confirmation = True  # Mark function

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[sensitive_action],
)
```

---

## Structured Input/Output

### Input Schema

**Configuration:**
```python
from pydantic import BaseModel, Field

class SearchQuery(BaseModel):
    query: str = Field(..., description="Search query")
    filters: dict = Field(default={}, description="Search filters")
    max_results: int = Field(default=10, ge=1, le=100)

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    input_schema=SearchQuery,
)

# Validated input
result = agent.run(SearchQuery(query="AI trends", max_results=5))
```

### Output Schema

**Configuration:**
```python
from pydantic import BaseModel

class AnalysisResult(BaseModel):
    summary: str
    key_points: list[str]
    confidence: float
    sources: list[str]

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    output_schema=AnalysisResult,
)

result = agent.run("Analyze AI market trends")
analysis: AnalysisResult = result.content  # Typed, validated
```

---

## Reasoning and Advanced Features

### Reasoning Steps

**Configuration:**
```python
from agno.agent import Agent
from agno.reasoning.step import ReasoningSteps

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    reasoning=ReasoningSteps(),  # Enable step-by-step reasoning
    show_reasoning=True,          # Display reasoning
)
```

### Hooks (Pre/Post Processing)

**Configuration:**
```python
def pre_hook(agent, messages):
    """Called before LLM call."""
    print(f"Processing {len(messages)} messages")
    return messages

def post_hook(agent, response):
    """Called after LLM call."""
    print(f"Generated response: {len(response.content)} chars")
    return response

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    pre_hooks=[pre_hook],
    post_hooks=[post_hook],
)
```

### Evaluation

**Configuration:**
```python
from agno.eval.agent_as_judge import AgentAsJudge

evaluator = AgentAsJudge(
    model=OpenAIChat(id="gpt-4o"),
    evaluation_criteria=["accuracy", "relevance", "clarity"],
)

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    evals=[evaluator],
)
```

---

## Production Features (AgentOS)

### FastAPI Runtime

**Configuration:**
```python
from agno.agent import Agent
from agno.os import AgentOS
from agno.db.postgres import PostgresDb

# Create agents
agent = Agent(
    name="Production Agent",
    model=OpenAIChat(id="gpt-4o"),
    db=PostgresDb(db_url=os.getenv("DATABASE_URL")),
)

# Create AgentOS
agent_os = AgentOS(
    agents=[agent],
    db=PostgresDb(db_url=os.getenv("DATABASE_URL")),
)

# Get FastAPI app
app = agent_os.get_app()

# Run server
if __name__ == "__main__":
    agent_os.serve(app="main:app", host="0.0.0.0", port=7777)
```

**Features:**
- Pre-built REST and SSE endpoints
- Session management
- Authentication and RBAC
- Monitoring and tracing
- WebSocket support

### Control Plane UI

1. Start your AgentOS runtime
2. Visit https://os.agno.com
3. Add your runtime URL (e.g., http://localhost:7777)
4. Connect directly from browser (no proxy)

**Features:**
- Chat interface
- Session explorer
- Memory viewer
- Knowledge management
- Trace debugging
- Performance metrics

---

## Cursor IDE Configuration

### Setup Instructions

**Step 1: Add Agno Documentation**

1. Open Cursor Settings
2. Go to **Settings → Indexing & Docs**
3. Click **Add New Doc**
4. Add: `https://docs.agno.com/llms-full.txt`

**Step 2: Configure .cursorrules**

Create `.cursorrules` in your project root:

```
You are an expert in Python, Agno framework, and AI agent development.

Core Rules
- NEVER create agents in loops - reuse them for performance
- Always use output_schema for structured responses
- PostgreSQL in production, SQLite for dev only
- Start with single agent, scale up only when needed

Basic Agent (start here):
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

When to Use Each Pattern
- Single Agent (90%): One clear task or domain
- Team (autonomous): Multiple specialized agents, complex tasks
- Workflow (programmatic): Sequential steps with clear flow

Common Mistakes
- Creating agents in loops (massive performance hit)
- Using Team when single agent would work
- Forgetting search_knowledge=True with knowledge
- Using SQLite in production
- Not adding history when context matters

Production
- Use PostgresDb not SqliteDb
- Set show_tool_calls=False, debug_mode=False
- Wrap agent.run() in try-except

Docs: https://docs.agno.com
```

**Step 3: Project Structure**

```
your-project/
├── .cursorrules          # Cursor configuration
├── .env                  # API keys
├── requirements.txt      # Dependencies
├── agents/              # Agent definitions
│   ├── __init__.py
│   ├── research_agent.py
│   └── writer_agent.py
├── teams/               # Team configurations
│   └── content_team.py
├── workflows/           # Workflow definitions
│   └── pipeline.py
└── main.py             # AgentOS entrypoint
```

**Step 4: Environment Variables**

Create `.env` file:
```bash
# LLM API Keys
OPENAI_API_KEY=your-key
ANTHROPIC_API_KEY=your-key
GOOGLE_API_KEY=your-key

# Database
DATABASE_URL=postgresql://localhost/agno_prod

# Agno
AGNO_TELEMETRY=false
```

**Step 5: Quick Start Template**

```python
# main.py
from agno.agent import Agent
from agno.models.openai import OpenAIChat
from agno.db.sqlite import SqliteDb
from agno.tools.duckduckgo import DuckDuckGoTools

agent = Agent(
    name="My Agent",
    model=OpenAIChat(id="gpt-4o"),
    tools=[DuckDuckGoTools()],
    db=SqliteDb(db_file="tmp/agent.db"),
    add_history_to_context=True,
    markdown=True,
)

if __name__ == "__main__":
    agent.print_response("Your query", stream=True)
```

### Cursor AI Features with Agno

**1. Code Generation**
- Ask Cursor to generate agents: "Create a research agent with web search"
- Cursor will use Agno patterns from .cursorrules

**2. Documentation Lookup**
- Cursor can reference docs.agno.com directly
- Press Cmd+K and ask about specific features

**3. Debugging**
- Cursor understands Agno's error patterns
- Can suggest fixes based on framework docs

**4. Autocomplete**
- Full autocomplete for Agno classes and methods
- Type hints from Pydantic schemas

---

## Performance Best Practices

### Critical Rules

1. **NEVER create agents in loops** (529× slower)
2. **Use PostgreSQL in production** (not SQLite)
3. **Enable search_knowledge=True** for RAG
4. **Add history when context matters**
5. **Use output_schema for structured data**
6. **Start with single agent, scale up only when needed**

### Optimization Checklist

- [ ] Agents created once and reused
- [ ] PostgreSQL for production database
- [ ] Session summaries enabled for long conversations
- [ ] Context compression configured
- [ ] Appropriate num_history_runs value
- [ ] Knowledge base indexed and optimized
- [ ] Tool calls optimized (parallel execution)
- [ ] Debug mode disabled in production
- [ ] show_tool_calls disabled in production
- [ ] Error handling and retries configured

---

## Summary

The Agno framework provides:
- **Performance**: 529× faster than alternatives
- **Production-Ready**: FastAPI runtime with monitoring
- **Privacy-First**: Runs in your infrastructure
- **Type-Safe**: Pydantic schemas throughout
- **Flexible**: Single agent to complex multi-agent systems
- **Model-Agnostic**: Works with any LLM provider

For complete documentation, visit **https://docs.agno.com**
