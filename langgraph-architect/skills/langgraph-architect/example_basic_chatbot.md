# Basic Chatbot

Implementation example of a basic chatbot using LangGraph.

## Recommended Directory Structure

```
basic_chatbot/
├── pyproject.toml
├── .env                    # ANTHROPIC_API_KEY
├── src/
│   └── basic_chatbot/
│       ├── __init__.py
│       ├── main.py         # Entry point
│       ├── graph.py        # Graph construction
│       ├── nodes.py        # Node functions
│       └── config.py       # LLM configuration
└── tests/
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

### nodes.py

```python
"""Node functions"""
from langgraph.graph import MessagesState
from .config import LLM

def chatbot_node(state: MessagesState) -> dict:
    """Chatbot node"""
    response = LLM.invoke(state["messages"])
    return {"messages": [response]}
```

### graph.py

```python
"""Graph construction"""
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver

from .nodes import chatbot_node

def create_graph():
    """Build and compile the graph"""
    builder = StateGraph(MessagesState)
    builder.add_node("chatbot", chatbot_node)
    builder.add_edge(START, "chatbot")
    builder.add_edge("chatbot", END)

    checkpointer = MemorySaver()
    return builder.compile(checkpointer=checkpointer)

graph = create_graph()
```

### main.py

```python
"""Entry point"""
from .graph import graph

def main():
    config = {"configurable": {"thread_id": "conversation-1"}}

    while True:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            break

        for chunk in graph.stream(
            {"messages": [{"role": "user", "content": user_input}]},
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
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.checkpoint.memory import MemorySaver
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-sonnet-4-5-20250929")

def chatbot_node(state: MessagesState):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node("chatbot", chatbot_node)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)

if __name__ == "__main__":
    config = {"configurable": {"thread_id": "conversation-1"}}
    while True:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            break
        for chunk in graph.stream(
            {"messages": [{"role": "user", "content": user_input}]},
            config,
            stream_mode="values"
        ):
            chunk["messages"][-1].pretty_print()
```

## Explanation

### 1. MessagesState

```python
from langgraph.graph import MessagesState

# MessagesState is equivalent to:
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

- `messages`: List of messages
- `add_messages`: Reducer that adds new messages

### 2. Checkpointer

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```

- Saves conversation state
- Continues conversation with same `thread_id`

### 3. Streaming

```python
for chunk in graph.stream(input, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

- `stream_mode="values"`: Complete state after each step
- `pretty_print()`: Displays messages in a readable format

## Extension Examples

### Adding System Message

```python
def chatbot_with_system(state: MessagesState):
    """With system message"""
    system_msg = {
        "role": "system",
        "content": "You are a helpful assistant."
    }

    response = llm.invoke([system_msg] + state["messages"])
    return {"messages": [response]}
```

### Limiting Message History

```python
def chatbot_with_limit(state: MessagesState):
    """Use only the latest 10 messages"""
    recent_messages = state["messages"][-10:]
    response = llm.invoke(recent_messages)
    return {"messages": [response]}
```

## Related Pages

- [01_core_concepts_overview.md](01_core_concepts_overview.md) - Understanding fundamental concepts
- [03_memory_management_overview.md](03_memory_management_overview.md) - Checkpointer details
- [example_rag_agent.md](example_rag_agent.md) - More advanced example
