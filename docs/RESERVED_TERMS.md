# Agno Framework Reserved Terms Glossary

This document defines all reserved terms, keywords, and concepts used in the Agno framework.

**Based on official documentation at https://docs.agno.com**

---

## Core Entities

### Agent
**Type:** Class  
**Module:** `agno.agent`

An autonomous entity that processes input using an LLM, executes tools, and produces output.

**Key Attributes:**
- `name`: Agent identifier (optional)
- `model`: LLM model to use
- `instructions`: System message/behavior guide
- `tools`: List of available tools/functions
- `knowledge`: Knowledge base for RAG
- `memory`: Memory manager for user preferences
- `db`: Database for persistence
- `session_id`: Current session identifier
- `user_id`: Current user identifier

**Example:**
```python
agent = Agent(
    name="Research Agent",
    model=OpenAIChat(id="gpt-4o"),
    instructions="You are a helpful research assistant",
)
```

---

### Team
**Type:** Class  
**Module:** `agno.team.team`

A collection of agents that collaborate autonomously. A leader LLM decides which agent(s) to delegate tasks to.

**Key Attributes:**
- `name`: Team identifier
- `members`: List of agent members
- `model`: Leader LLM for coordination
- `instructions`: Team-level behavior guide
- `db`: Shared database
- `session_id`: Team session identifier

**Example:**
```python
team = Team(
    name="Content Team",
    members=[researcher, writer, editor],
    model=OpenAIChat(id="gpt-4o"),
    instructions="Coordinate to produce high-quality content",
)
```

**Use When:**
- Multiple specialized agents needed
- Task requires different perspectives
- Autonomous coordination preferred

---

### Workflow
**Type:** Class  
**Module:** `agno.workflow.workflow`

Pipeline-based orchestration with programmatic control over agent execution order and logic.

**Key Attributes:**
- `name`: Workflow identifier
- `steps`: Function or list of steps to execute
- `db`: Database for state persistence
- `session_id`: Workflow session identifier

**Example:**
```python
async def pipeline(workflow, input_data):
    result1 = await agent1.arun(input_data)
    result2 = await agent2.arun(result1.content)
    return result2.content

workflow = Workflow(
    name="Data Pipeline",
    steps=pipeline,
)
```

**Use When:**
- Sequential processing required
- Conditional branching needed
- Full control over execution flow

---

## Session Management

### session_id
**Type:** String (UUID)  
**Scope:** Session-level

Unique identifier that groups related runs together, representing a multi-turn conversation.

**Behavior:**
- Auto-generated if not provided
- Persists across runs
- Isolates conversation context
- Required for history and state management

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    session_id="session-123",  # Explicit session
)

# Auto-generated if omitted
agent = Agent(model=OpenAIChat(id="gpt-4o"))
# agent.session_id will be auto-created
```

---

### run_id
**Type:** String (UUID)  
**Scope:** Run-level

Unique identifier for each individual interaction with an agent/team/workflow.

**Behavior:**
- Auto-generated for each run
- Used for tracing and debugging
- Links to specific run history
- Returned in RunOutput

**Example:**
```python
result = agent.run("Hello")
print(result.run_id)  # e.g., "run-456def"
```

---

### user_id
**Type:** String  
**Scope:** User-level

Associates sessions and state with a specific user. **Required for memory and multi-user isolation.**

**Behavior:**
- Isolates user data
- Required for user memory
- Enables personalization
- Ensures privacy in multi-user systems

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    user_id="user-789",  # Required for memory
    memory=MemoryManager(),
)
```

**Security Note:**
- Critical for session isolation (CVE-2025-64168)
- Prevents data leakage between users
- Always set in multi-user applications

---

### agent_id
**Type:** String  
**Scope:** Agent-level

Identifies the specific agent instance managing a session.

**Example:**
```python
agent = Agent(
    id="research-agent-001",
    name="Research Agent",
    model=OpenAIChat(id="gpt-4o"),
)
```

---

## State and Context

### session_state
**Type:** Dictionary  
**Scope:** Session-level

Structured data that persists across all runs within a session. Think of it as a "session-level database."

**Behavior:**
- Persists in database
- Survives restarts
- Mutable across runs
- Isolated per session

**Priority Order:**
1. Provided in `run()` call (highest)
2. Loaded from database
3. Agent's default state (lowest)

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    session_state={"shopping_list": [], "budget": 1000},
)

# State persists across runs
agent.run("Add milk to shopping list")
agent.run("What's on my list?")  # Remembers milk

# Access programmatically
session = agent.get_session()
print(session.session_state["shopping_list"])
```

---

### instructions
**Type:** String  
**Scope:** Agent/Team/Workflow-level

System message that defines agent behavior, personality, and guidelines.

**Alternative Names:**
- `system_message` (deprecated, use `instructions`)

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    instructions="""
    You are a financial analyst. 
    Always cite sources.
    Use tables for comparisons.
    No speculation, facts only.
    """,
)
```

**Best Practices:**
- Be specific and detailed
- Include format preferences
- Specify constraints
- Define tone and style

---

### guidelines
**Type:** String  
**Scope:** Agent-level  
**Status:** Not a reserved keyword (use `instructions` instead)

Note: While "guidelines" is commonly used in documentation, the actual parameter is `instructions`.

---

## Memory and Knowledge

### Memory
**Type:** Class  
**Module:** `agno.memory`

System for storing and retrieving user-specific facts and preferences across sessions.

**Key Components:**
- `MemoryManager`: Manages memory capture and retrieval
- User memories stored in database
- Associated with `user_id`

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    memory=MemoryManager(),
    user_id="user-123",  # Required
)
```

**Stored Data:**
- User preferences
- Facts learned about user
- Personalization data

---

### Knowledge
**Type:** Class  
**Module:** `agno.knowledge.knowledge`

Document store with semantic search for Retrieval-Augmented Generation (RAG).

**Key Components:**
- `vector_db`: Vector database for embeddings
- `contents_db`: Optional metadata storage
- Document readers and chunking strategies

**Example:**
```python
knowledge = Knowledge(
    vector_db=LanceDb(
        uri="tmp/lancedb",
        table_name="documents",
        search_type=SearchType.hybrid,
    ),
)

agent = Agent(
    knowledge=knowledge,
    search_knowledge=True,  # CRITICAL: enables RAG
)
```

---

### Culture
**Type:** Class  
**Module:** `agno.culture.manager`

Shared, organization-wide memory that persists across all agents and users.

**Key Attributes:**
- `db`: Shared database
- `model`: LLM for culture management

**Example:**
```python
culture = CultureManager(
    db=PostgresDb(db_url="..."),
    model=OpenAIChat(id="gpt-4o"),
)

# All agents share culture
agent1 = Agent(culture=culture, ...)
agent2 = Agent(culture=culture, ...)
```

**Use Cases:**
- Organization policies
- Shared conventions
- Team knowledge base

---

## History and Context

### add_history_to_context
**Type:** Boolean  
**Default:** False

Controls whether previous conversation messages are included in context.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    add_history_to_context=True,  # Include previous messages
    num_history_runs=5,            # Last 5 conversations
)
```

---

### num_history_runs
**Type:** Integer  
**Default:** None (all history)

Number of previous runs to include in context when `add_history_to_context=True`.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    add_history_to_context=True,
    num_history_runs=3,  # Only last 3 runs
)
```

---

### search_knowledge
**Type:** Boolean  
**Default:** False

Enables agentic RAG - agent decides when to search knowledge base.

**CRITICAL:** Must be `True` for knowledge base to be used.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    knowledge=knowledge_base,
    search_knowledge=True,  # Agent can search knowledge
)
```

---

## Input/Output Schemas

### input_schema
**Type:** Pydantic Model  
**Scope:** Agent/Team/Workflow-level

Defines and validates the structure of input data.

**Example:**
```python
class QueryInput(BaseModel):
    question: str
    context: Optional[str] = None

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    input_schema=QueryInput,
)

# Validated input
result = agent.run(QueryInput(question="What is AI?"))
```

---

### output_schema
**Type:** Pydantic Model  
**Scope:** Agent/Team/Workflow-level

Defines and validates the structure of output data. Ensures structured, type-safe responses.

**Example:**
```python
class AnalysisOutput(BaseModel):
    summary: str
    confidence: float
    sources: List[str]

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    output_schema=AnalysisOutput,
)

result = agent.run("Analyze this")
output: AnalysisOutput = result.content  # Typed and validated
```

---

## Tools and Functions

### tools
**Type:** List  
**Scope:** Agent/Team-level

List of tools, toolkits, or functions the agent can use.

**Example:**
```python
from agno.tools.duckduckgo import DuckDuckGoTools

def custom_tool(arg: str) -> str:
    """Custom tool description."""
    return f"Result: {arg}"

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[
        DuckDuckGoTools(),
        custom_tool,
    ],
)
```

---

### show_tool_calls
**Type:** Boolean  
**Default:** False

Controls whether tool execution details are displayed.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    show_tool_calls=True,  # Show tool execution
)
```

**Production Best Practice:** Set to `False` in production.

---

## Database and Storage

### db
**Type:** BaseDb or AsyncBaseDb  
**Scope:** Agent/Team/Workflow-level

Database connection for persisting sessions, history, memory, and state.

**Supported Databases:**
- PostgreSQL (production - required)
- SQLite (development only)
- MySQL
- MongoDB
- DynamoDB
- Redis
- In-Memory (testing only)

**Example:**
```python
from agno.db.postgres import PostgresDb

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    db=PostgresDb(db_url="postgresql://localhost/prod"),
)
```

---

## Guardrails

### guardrails
**Type:** List[BaseGuardrail]  
**Scope:** Agent/Team-level

List of validation and safety checks applied to input and output.

**Example:**
```python
from agno.guardrails.pii import PIIGuardrail
from agno.guardrails.prompt_injection import PromptInjectionGuardrail

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    guardrails=[
        PIIGuardrail(),
        PromptInjectionGuardrail(),
    ],
)
```

---

## Human-in-the-Loop

### require_human_confirmation
**Type:** Boolean  
**Default:** False

Requires user approval before executing any tool.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    tools=[EmailTools()],
    require_human_confirmation=True,  # Approval required
)
```

---

## Workflow-Specific Terms

### Step
**Type:** Class  
**Module:** `agno.workflow.step`

A single step in a workflow.

**Example:**
```python
from agno.workflow.step import Step

step1 = Step(name="Process", fn=process_function)
```

---

### Condition
**Type:** Class  
**Module:** `agno.workflow.condition`

Conditional branching in workflows.

**Example:**
```python
from agno.workflow.condition import Condition

condition = Condition(
    name="Check Result",
    condition=lambda state: state["score"] > 0.8,
    true_step=success_step,
    false_step=retry_step,
)
```

---

### Loop
**Type:** Class  
**Module:** `agno.workflow.loop`

Repeat steps until a condition is met.

**Example:**
```python
from agno.workflow.loop import Loop

loop = Loop(
    name="Retry Loop",
    step=process_step,
    condition=lambda state: state["success"] == True,
    max_iterations=3,
)
```

---

### Parallel
**Type:** Class  
**Module:** `agno.workflow.parallel`

Execute multiple steps concurrently.

**Example:**
```python
from agno.workflow.parallel import Parallel

parallel = Parallel(
    name="Parallel Processing",
    steps=[step1, step2, step3],
)
```

---

### Router
**Type:** Class  
**Module:** `agno.workflow.router`

Route to different steps based on conditions.

---

## AgentOS Terms

### AgentOS
**Type:** Class  
**Module:** `agno.os`

Production runtime that wraps agents/teams/workflows in a FastAPI application.

**Example:**
```python
from agno.os import AgentOS

agent_os = AgentOS(
    agents=[agent1, agent2],
    db=PostgresDb(db_url="..."),
)

app = agent_os.get_app()  # FastAPI app
```

---

## Execution Methods

### run()
**Type:** Method  
**Returns:** RunOutput

Synchronous execution of agent/team/workflow.

**Example:**
```python
result = agent.run("Hello, world!")
print(result.content)
```

---

### arun()
**Type:** Async Method  
**Returns:** RunOutput

Asynchronous execution.

**Example:**
```python
result = await agent.arun("Hello, world!")
```

---

### run_stream()
**Type:** Method  
**Returns:** Iterator[RunOutputEvent]

Synchronous streaming execution.

**Example:**
```python
for chunk in agent.run_stream("Write an essay"):
    print(chunk.content, end="", flush=True)
```

---

### arun_stream()
**Type:** Async Method  
**Returns:** AsyncIterator[RunOutputEvent]

Asynchronous streaming execution.

**Example:**
```python
async for chunk in agent.arun_stream("Write an essay"):
    print(chunk.content, end="", flush=True)
```

---

### print_response()
**Type:** Method  
**Returns:** None

Convenience method that runs and prints the response.

**Example:**
```python
agent.print_response("Your query", stream=True)
```

---

## Response Types

### RunOutput
**Type:** Class  
**Module:** `agno.run.agent`

Response object from agent/team/workflow execution.

**Key Attributes:**
- `content`: Response content (str or output_schema instance)
- `run_id`: Unique run identifier
- `session_id`: Session identifier
- `messages`: Conversation messages
- `metrics`: Performance metrics

---

### RunOutputEvent
**Type:** Class  
**Module:** `agno.run.agent`

Streaming event from agent/team/workflow execution.

**Key Attributes:**
- `content`: Partial content
- `event_type`: Type of event
- `run_id`: Run identifier

---

## Model Configuration

### Model
**Type:** Base Class  
**Module:** `agno.models.base`

Base class for all LLM model integrations.

**Common Parameters:**
- `id`: Model identifier
- `temperature`: Randomness (0.0-1.0)
- `max_tokens`: Maximum response length
- `api_key`: API key (or use environment variables)

---

## Debugging and Monitoring

### debug_mode
**Type:** Boolean  
**Default:** False

Enables verbose logging for debugging.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    debug_mode=True,  # Verbose logging
)
```

**Production Best Practice:** Set to `False` in production.

---

### markdown
**Type:** Boolean  
**Default:** False

Formats output as markdown.

**Example:**
```python
agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    markdown=True,  # Markdown output
)
```

---

## Compression and Optimization

### CompressionManager
**Type:** Class  
**Module:** `agno.compression.manager`

Manages context compression to reduce token usage.

**Example:**
```python
from agno.compression.manager import CompressionManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    compression_manager=CompressionManager(
        compress_messages=True,
        compression_ratio=0.5,
    ),
)
```

---

### SessionSummaryManager
**Type:** Class  
**Module:** `agno.session.summary`

Manages session summaries to condense long conversations.

**Example:**
```python
from agno.session.summary import SessionSummaryManager

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    session_summary_manager=SessionSummaryManager(
        num_runs_to_summarize=10,
    ),
)
```

---

## Hooks

### pre_hooks
**Type:** List[Callable]

Functions called before LLM execution.

**Example:**
```python
def log_input(agent, messages):
    print(f"Processing {len(messages)} messages")
    return messages

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    pre_hooks=[log_input],
)
```

---

### post_hooks
**Type:** List[Callable]

Functions called after LLM execution.

**Example:**
```python
def log_output(agent, response):
    print(f"Generated {len(response.content)} chars")
    return response

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    post_hooks=[log_output],
)
```

---

## Evaluation

### evals
**Type:** List[BaseEval]

List of evaluation functions to assess agent performance.

**Example:**
```python
from agno.eval.agent_as_judge import AgentAsJudge

agent = Agent(
    model=OpenAIChat(id="gpt-4o"),
    evals=[
        AgentAsJudge(
            model=OpenAIChat(id="gpt-4o"),
            evaluation_criteria=["accuracy", "relevance"],
        )
    ],
)
```

---

## Media Support

### images, audios, videos, files
**Type:** List  
**Scope:** run() method parameters

Multimodal inputs for agent processing.

**Example:**
```python
from agno.media import Image, Audio, Video, File

result = agent.run(
    "Analyze this content",
    images=[Image.from_path("photo.jpg")],
    audios=[Audio.from_path("speech.mp3")],
    videos=[Video.from_path("clip.mp4")],
    files=[File.from_path("document.pdf")],
)
```

---

## Summary of Critical Terms

### Must-Know for Basic Usage
- `Agent` - Core building block
- `instructions` - Defines agent behavior
- `tools` - Functions agent can use
- `run()` / `arun()` - Execute agent
- `db` - Database for persistence
- `session_id` - Session identifier

### Must-Know for Persistence
- `session_state` - Structured session data
- `add_history_to_context` - Include chat history
- `user_id` - User identifier (required for memory)
- `memory` - User preference storage

### Must-Know for RAG
- `Knowledge` - Document store
- `search_knowledge` - Enable knowledge search
- `vector_db` - Vector database for embeddings

### Must-Know for Teams
- `Team` - Multi-agent collaboration
- `members` - List of agents in team

### Must-Know for Workflows
- `Workflow` - Orchestration pipeline
- `steps` - Workflow steps
- `session_state` - Shared workflow state

### Must-Know for Production
- `PostgresDb` - Production database
- `AgentOS` - Production runtime
- `guardrails` - Input/output validation
- `debug_mode=False` - Disable in production
- `show_tool_calls=False` - Disable in production

---

## References

- Official Documentation: https://docs.agno.com
- API Reference: https://docs.agno.com/reference
- Examples: https://github.com/agno-agi/agno/tree/main/cookbook
