# RAG Agent

Implementation example of a RAG (Retrieval-Augmented Generation) agent with search functionality.

## Recommended Directory Structure

```
rag_agent/
├── pyproject.toml
├── .env                    # ANTHROPIC_API_KEY
├── src/
│   └── rag_agent/
│       ├── __init__.py
│       ├── main.py         # Entry point
│       ├── graph.py        # Graph construction
│       ├── nodes.py        # Node functions
│       ├── edges.py        # Conditional edge functions
│       ├── config.py       # LLM configuration
│       └── tools/
│           ├── __init__.py
│           └── search.py   # Search tools
└── tests/
    ├── test_tools.py
    └── test_graph.py
```

## Modular Code

### config.py

```python
"""LLM configuration"""
import os
from dotenv import load_dotenv
from langchain_anthropic import ChatAnthropic

load_dotenv()

LLM = ChatAnthropic(
    model="claude-sonnet-4-5-20250929",
    api_key=os.getenv("ANTHROPIC_API_KEY")
)
```

### tools/search.py

```python
"""Search tools"""
from langchain_core.tools import tool

@tool
def retrieve_documents(query: str) -> str:
    """Retrieve relevant documents.

    Args:
        query: Search query
    """
    # In practice, search with vector store, etc.
    docs = [
        "LangGraph is an agent framework.",
        "StateGraph manages state.",
        "You can extend agents with tools."
    ]
    return "\n".join(docs)
```

### tools/__init__.py

```python
"""Tool exports"""
from .search import retrieve_documents

ALL_TOOLS = [retrieve_documents]
```

### nodes.py

```python
"""Node functions"""
from langgraph.graph import MessagesState
from .config import LLM
from .tools import ALL_TOOLS

llm_with_tools = LLM.bind_tools(ALL_TOOLS)

def agent_node(state: MessagesState) -> dict:
    """Agent node"""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}
```

### edges.py

```python
"""Conditional edge functions"""
from typing import Literal
from langgraph.graph import MessagesState

def should_continue(state: MessagesState) -> Literal["tools", "end"]:
    """Determine tool usage"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return "end"
```

### graph.py

```python
"""Graph construction"""
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

from .nodes import agent_node
from .edges import should_continue
from .tools import ALL_TOOLS

def create_graph():
    """Build and compile the graph"""
    builder = StateGraph(MessagesState)

    # Add nodes
    builder.add_node("agent", agent_node)
    builder.add_node("tools", ToolNode(ALL_TOOLS))

    # Add edges
    builder.add_edge(START, "agent")
    builder.add_conditional_edges(
        "agent",
        should_continue,
        {"tools": "tools", "end": END}
    )
    builder.add_edge("tools", "agent")

    # Compile
    checkpointer = MemorySaver()
    return builder.compile(checkpointer=checkpointer)

graph = create_graph()
```

### main.py

```python
"""Entry point"""
from .graph import graph

def main():
    config = {"configurable": {"thread_id": "rag-session-1"}}

    query = "What is LangGraph?"

    for chunk in graph.stream(
        {"messages": [{"role": "user", "content": query}]},
        config,
        stream_mode="values"
    ):
        chunk["messages"][-1].pretty_print()

if __name__ == "__main__":
    main()
```

## Single File Version (for learning)

```python
"""Single file version - for learning purposes only"""
from typing import Literal
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool

@tool
def retrieve_documents(query: str) -> str:
    """Retrieve relevant documents."""
    docs = [
        "LangGraph is an agent framework.",
        "StateGraph manages state.",
        "You can extend agents with tools."
    ]
    return "\n".join(docs)

tools = [retrieve_documents]
llm = ChatAnthropic(model="claude-sonnet-4-5-20250929")
llm_with_tools = llm.bind_tools(tools)

def agent_node(state: MessagesState):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def should_continue(state: MessagesState) -> Literal["tools", "end"]:
    if state["messages"][-1].tool_calls:
        return "tools"
    return "end"

builder = StateGraph(MessagesState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", should_continue, {"tools": "tools", "end": END})
builder.add_edge("tools", "agent")

graph = builder.compile(checkpointer=MemorySaver())
```

## Execution Flow

```
User Query: "What is LangGraph?"
    ↓
[Agent Node]
    ↓
LLM: "I'll search for information" + ToolCall(retrieve_documents)
    ↓
[Tool Node] ← Execute search
    ↓
ToolMessage: "LangGraph is an agent framework..."
    ↓
[Agent Node] ← Use search results
    ↓
LLM: "LangGraph is a framework for building agents..."
    ↓
END
```

## Extension Examples

### Multiple Search Tools

```python
@tool
def web_search(query: str) -> str:
    """Search the web"""
    return search_web(query)

@tool
def database_search(query: str) -> str:
    """Search database"""
    return search_database(query)

tools = [retrieve_documents, web_search, database_search]
```

### Vector Search Implementation

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# Initialize vector store
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(
    ["LangGraph is an agent framework.", ...],
    embeddings
)

@tool
def semantic_search(query: str) -> str:
    """Perform semantic search"""
    docs = vectorstore.similarity_search(query, k=3)
    return "\n".join([doc.page_content for doc in docs])
```

### Adding Human-in-the-Loop

```python
from langgraph.types import interrupt

@tool
def sensitive_search(query: str) -> str:
    """Search sensitive information (requires approval)"""
    approved = interrupt({
        "action": "sensitive_search",
        "query": query,
        "message": "Approve this sensitive search?"
    })

    if approved:
        return perform_sensitive_search(query)
    else:
        return "Search cancelled by user"
```

## Related Pages

- [02_graph_architecture_agent.md](02_graph_architecture_agent.md) - Agent pattern
- [04_tool_integration_overview.md](04_tool_integration_overview.md) - Tool details
- [example_basic_chatbot.md](example_basic_chatbot.md) - Basic chatbot
