# Google Gemini Model IDs

List of available model IDs for the Google Gemini API.

> **Last Updated**: 2025-11-24

## Model List

While there are many models available, `gemini-2.5-flash` is generally recommended for development at this time. It offers a good balance of cost and performance for a wide range of use cases.

### Gemini 3.x (Latest)

| Model ID                                | Context | Max Output | Use Case                    |
| ---------------------------------------- | ------------ | -------- | ------------------ |
| `google/gemini-3-pro-preview`            | -            | 64K      | Latest high-performance model |
| `google/gemini-3-pro-image-preview`      | -            | -        | Image generation           |
| `google/gemini-3-pro-image-preview-edit` | -            | -        | Image editing           |

### Gemini 2.5

| Model ID               | Context | Max Output | Use Case                   |
| ----------------------- | ------------ | -------- | ---------------------- |
| `google/gemini-2.5-pro` | 1M (2M planned) | -        | High performance                 |
| `gemini-2.5-flash`      | 1M           | -        | Fast balanced model (recommended) |
| `gemini-2.5-flash-lite` | 1M           | -        | Lightweight and fast               |

**Note**: Free tier is limited to approximately 32K tokens. Gemini Advanced (2.5 Pro) supports 1M tokens.

### Gemini 2.0

| Model ID          | Context | Max Output | Use Case   |
| ------------------ | ------------ | -------- | ------ |
| `gemini-2.0-flash` | 1M           | -        | Stable version |

## Basic Usage

```python
from langchain_google_genai import ChatGoogleGenerativeAI

# Recommended: Balanced model
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash")

# Also works with prefix
llm = ChatGoogleGenerativeAI(model="models/gemini-2.5-flash")

# High-performance version
llm = ChatGoogleGenerativeAI(model="google/gemini-3-pro")

# Lightweight version
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
```

### Environment Variables

```bash
export GOOGLE_API_KEY="your-api-key"
```

## Model Selection Guide

| Use Case               | Recommended Model                     |
| ------------------ | ------------------------------ |
| Cost-focused         | `gemini-2.5-flash-lite`        |
| Balanced           | `gemini-2.5-flash`             |
| Performance-focused           | `google/gemini-3-pro`          |
| Large context | `gemini-2.5-pro` (1M tokens) |

## Gemini Features

### 1. Large Context Window

Gemini is the **industry's first model to support 1M tokens**:

| Tier                    | Context Limit |
| ------------------------- | ---------------- |
| Gemini Advanced (2.5 Pro) | 1M tokens      |
| Vertex AI                 | 1M tokens      |
| Free tier                    | ~32K tokens  |

**Use Cases**:

- Long document analysis
- Understanding entire codebases
- Long conversation history

```python
# Processing large context
llm = ChatGoogleGenerativeAI(
    model="gemini-2.5-pro",
    max_tokens=8192  # Specify output token count
)
```

**Future**: Gemini 2.5 Pro is planned to support 2M token context windows.

### 2. Multimodal Support

Image input and generation capabilities (see [Advanced Features](06_llm_model_ids_gemini_advanced.md) for details).

## Grounding (Google Search)

Grounding enables Gemini to access real-time information via Google Search, reducing hallucinations and providing verifiable sources.

### Supported Models

| Model | Google Search Grounding | Google Maps Grounding |
| ----- | ----------------------- | --------------------- |
| `google/gemini-3-pro` | ✅ | ❌ |
| `google/gemini-2.5-pro` | ✅ | ✅ |
| `gemini-2.5-flash` | ✅ | ✅ |
| `gemini-2.5-flash-lite` | ✅ | ✅ |
| `gemini-2.0-flash` | ✅ | - |
| `gemini-2.0-flash-lite` | ❌ | - |

### Basic Usage

```python
from google import genai
from google.genai import types

client = genai.Client()

# Define grounding tool
grounding_tool = types.Tool(
    google_search=types.GoogleSearch()
)

config = types.GenerateContentConfig(
    tools=[grounding_tool]
)

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="What are the latest AI news today?",
    config=config,
)

print(response.text)
```

### Response Metadata

Grounded responses include `groundingMetadata`:

```python
# Access grounding metadata
metadata = response.candidates[0].grounding_metadata

# Search queries used
print(metadata.web_search_queries)

# Source information
for chunk in metadata.grounding_chunks:
    print(f"Source: {chunk.web.title} - {chunk.web.uri}")

# Text-to-source mappings (for inline citations)
for support in metadata.grounding_supports:
    print(f"Segment: {support.segment.text}")
    print(f"Source indices: {support.grounding_chunk_indices}")
```

### Use Cases

- **Real-time information**: Current events, stock prices, weather
- **Fact verification**: Reduce hallucinations with verifiable sources
- **Research assistance**: Provide citations and references

### Pricing Notes

- Billed per search query executed by the model
- Multiple searches in one request = multiple billable uses
- Model automatically determines when search improves the answer

### Reference

- [Grounding Documentation](https://ai.google.dev/gemini-api/docs/grounding)

## Important Notes

- ❌ **Deprecated**: Gemini 1.0, 1.5 series are no longer available
- ✅ **Migration Recommended**: Use `gemini-2.5-flash` or later models

## Detailed Documentation

For advanced configuration and multimodal features, see:

- **[Gemini Advanced Features](06_llm_model_ids_gemini_advanced.md)**

## Reference Links

- [Gemini API Official](https://ai.google.dev/gemini-api/docs/models)
- [Google AI Studio](https://makersuite.google.com/)
- [LangChain Integration](https://docs.langchain.com/oss/python/integrations/chat/google_generative_ai)
