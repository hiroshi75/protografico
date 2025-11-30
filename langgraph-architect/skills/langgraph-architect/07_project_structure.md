# Project Structure

LangGraphアプリケーションの推奨ディレクトリ構成とファイル分割のベストプラクティス。

## 基本原則

**main.pyに全てを書かない**。理解しやすさ・保守性・テスト容易性のために、役割ごとにファイルを分割する。

## 推奨ディレクトリ構成

### シンプルなアプリケーション

```
my_agent/
├── pyproject.toml          # プロジェクト設定
├── .env                    # 環境変数（API キー等）
├── src/
│   └── my_agent/
│       ├── __init__.py
│       ├── main.py         # エントリーポイント（実行のみ）
│       ├── graph.py        # グラフ構築
│       ├── state.py        # State定義
│       ├── nodes.py        # ノード関数
│       └── config.py       # LLM・設定の初期化
└── tests/
    ├── __init__.py
    └── test_graph.py
```

### 中規模アプリケーション

```
my_agent/
├── pyproject.toml
├── .env
├── src/
│   └── my_agent/
│       ├── __init__.py
│       ├── main.py             # エントリーポイント
│       ├── graph.py            # メイングラフ構築
│       ├── state.py            # State定義
│       ├── config.py           # LLM・設定
│       ├── nodes/              # ノード関数（機能別）
│       │   ├── __init__.py
│       │   ├── agent.py        # エージェントノード
│       │   ├── router.py       # ルーティングノード
│       │   └── processor.py    # 処理ノード
│       ├── tools/              # ツール定義
│       │   ├── __init__.py
│       │   ├── search.py
│       │   └── database.py
│       └── subgraphs/          # サブグラフ
│           ├── __init__.py
│           └── rag.py
└── tests/
    ├── __init__.py
    ├── test_nodes.py
    ├── test_tools.py
    └── test_graph.py
```

### 大規模アプリケーション

```
my_agent/
├── pyproject.toml
├── .env
├── src/
│   └── my_agent/
│       ├── __init__.py
│       ├── main.py
│       ├── graph.py
│       ├── state/              # State（複数ある場合）
│       │   ├── __init__.py
│       │   ├── base.py
│       │   └── messages.py
│       ├── config/             # 設定
│       │   ├── __init__.py
│       │   ├── llm.py
│       │   └── settings.py
│       ├── nodes/
│       │   ├── __init__.py
│       │   └── ...
│       ├── edges/              # 条件付きエッジ
│       │   ├── __init__.py
│       │   └── routing.py
│       ├── tools/
│       │   ├── __init__.py
│       │   └── ...
│       ├── subgraphs/
│       │   ├── __init__.py
│       │   └── ...
│       └── memory/             # メモリ・永続化
│           ├── __init__.py
│           └── checkpointer.py
└── tests/
    └── ...
```

## 各ファイルの役割

### state.py - State定義

```python
"""State定義"""
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    """エージェントの状態"""
    messages: Annotated[list, add_messages]
    context: str
    iteration_count: int
```

### config.py - 設定・LLM初期化

```python
"""設定とLLM初期化"""
import os
from dotenv import load_dotenv
from langchain_anthropic import ChatAnthropic

load_dotenv()

# LLM設定
LLM = ChatAnthropic(
    model="claude-sonnet-4-5-20250929",
    api_key=os.getenv("ANTHROPIC_API_KEY")
)

# その他の設定
MAX_ITERATIONS = 10
```

### nodes.py - ノード関数

```python
"""ノード関数"""
from .state import AgentState
from .config import LLM

def agent_node(state: AgentState) -> dict:
    """エージェントノード"""
    response = LLM.invoke(state["messages"])
    return {"messages": [response]}

def process_node(state: AgentState) -> dict:
    """処理ノード"""
    # 処理ロジック
    return {"context": "processed"}
```

### tools/ - ツール定義

```python
# tools/search.py
"""検索ツール"""
from langchain_core.tools import tool

@tool
def search_documents(query: str) -> str:
    """ドキュメントを検索する

    Args:
        query: 検索クエリ
    """
    # 検索ロジック
    return "検索結果"
```

```python
# tools/__init__.py
"""ツール一覧"""
from .search import search_documents
from .database import query_database

ALL_TOOLS = [search_documents, query_database]
```

### graph.py - グラフ構築

```python
"""グラフ構築"""
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

from .state import AgentState
from .nodes import agent_node, process_node
from .tools import ALL_TOOLS

def create_graph():
    """グラフを構築してコンパイル"""
    builder = StateGraph(AgentState)

    # ノード追加
    builder.add_node("agent", agent_node)
    builder.add_node("tools", ToolNode(ALL_TOOLS))
    builder.add_node("process", process_node)

    # エッジ追加
    builder.add_edge(START, "agent")
    builder.add_conditional_edges(
        "agent",
        should_continue,
        {"tools": "tools", "process": "process", "end": END}
    )
    builder.add_edge("tools", "agent")
    builder.add_edge("process", END)

    # コンパイル
    checkpointer = MemorySaver()
    return builder.compile(checkpointer=checkpointer)

def should_continue(state: AgentState) -> str:
    """継続条件を判定"""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return "end"

# グラフインスタンス
graph = create_graph()
```

### main.py - エントリーポイント

```python
"""エントリーポイント"""
from .graph import graph

def main():
    config = {"configurable": {"thread_id": "session-1"}}

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

## ファイル分割の基準

### 分割すべき場合

| 状況 | 分割先 |
|------|--------|
| ノードが3つ以上 | `nodes/` ディレクトリ |
| ツールが2つ以上 | `tools/` ディレクトリ |
| サブグラフがある | `subgraphs/` ディレクトリ |
| 複数のStateがある | `state/` ディレクトリ |
| 設定が複雑 | `config/` ディレクトリ |

### 分割の判断フロー

```
コード量が100行を超えた？
    ├─ Yes → 役割ごとにファイル分割
    └─ No → 単一ファイルでOK

ノード数が3つ以上？
    ├─ Yes → nodes/ ディレクトリを作成
    └─ No → nodes.py で十分

ツール数が2つ以上？
    ├─ Yes → tools/ ディレクトリを作成
    └─ No → graph.py 内で定義でOK
```

## インポートパターン

### 相対インポート（推奨）

```python
# graph.py
from .state import AgentState
from .nodes import agent_node
from .tools import ALL_TOOLS
```

### __init__.py でエクスポート

```python
# src/my_agent/__init__.py
from .graph import graph
from .state import AgentState

__all__ = ["graph", "AgentState"]
```

## テスト構成

```python
# tests/test_nodes.py
"""ノードのユニットテスト"""
import pytest
from my_agent.nodes import agent_node
from my_agent.state import AgentState

def test_agent_node():
    state: AgentState = {
        "messages": [{"role": "user", "content": "Hello"}],
        "context": "",
        "iteration_count": 0
    }
    result = agent_node(state)
    assert "messages" in result
```

```python
# tests/test_graph.py
"""グラフ統合テスト"""
import pytest
from dotenv import load_dotenv
from my_agent.graph import graph

load_dotenv()

def test_graph_execution():
    config = {"configurable": {"thread_id": "test-1"}}
    result = graph.invoke(
        {"messages": [{"role": "user", "content": "Hello"}]},
        config
    )
    assert len(result["messages"]) > 1
```

## アンチパターン

### 避けるべき構成

```python
# main.py に全てを書く（NG）
from langgraph.graph import StateGraph, START, END
from langchain_anthropic import ChatAnthropic

# State定義
class State(TypedDict):
    ...

# LLM初期化
llm = ChatAnthropic(...)

# ツール定義
@tool
def search(...):
    ...

# ノード定義
def agent_node(...):
    ...

def process_node(...):
    ...

# グラフ構築
builder = StateGraph(State)
...

# 実行
if __name__ == "__main__":
    ...
```

### 問題点

1. **可読性が低い**: 数百行のファイルは理解が困難
2. **テストが困難**: 個別のノード・ツールをテストしにくい
3. **再利用が困難**: 部品を他のプロジェクトで使い回せない
4. **並行開発が困難**: 複数人での作業でコンフリクトが発生

## 関連ページ

- [02_graph_architecture_subgraph.md](02_graph_architecture_subgraph.md) - サブグラフの構成
- [04_tool_integration_overview.md](04_tool_integration_overview.md) - ツール統合
- [example_basic_chatbot.md](example_basic_chatbot.md) - 基本チャットボット（モジュラー版）
- [example_rag_agent.md](example_rag_agent.md) - RAGエージェント（モジュラー版）
