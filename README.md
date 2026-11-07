# langchain-cloro

This package contains the LangChain integration for [cloro.dev](https://cloro.dev/) - a unified API for monitoring multiple AI providers including Google Search, ChatGPT, Gemini, and Bing AI Mode.

## Installation

```bash
pip install langchain-cloro
```

## Setup

You'll need a cloro API key. Get one at [https://cloro.dev](https://cloro.dev/).

Set the API key as an environment variable:

```bash
export CLORO_API_KEY="your-api-key-here"
```

Or pass it directly when initializing:

```python
from langchain_cloro import CloroGoogleSearch

tool = CloroGoogleSearch(cloro_api_key="your-api-key-here")
```

## Available Tools

### CloroGoogleSearch

Extract structured data from Google Search results, including organic results, People Also Ask questions, related searches, and optional AI Overview data.

```python
from langchain_cloro import CloroGoogleSearch

tool = CloroGoogleSearch()

# Basic search
results = tool.invoke({"query": "best laptops for programming"})

# With AI Overview
results = tool.invoke({
    "query": "best laptops for programming",
    "include_aioverview": True,
    "aioverview_markdown": True
})

# Multiple pages
results = tool.invoke({
    "query": "python tutorials",
    "pages": 3,
    "country": "US",
    "device": "desktop"
})
```

**Parameters:**
- `query` (str, required): The search query
- `country` (str): ISO 3166-1 alpha-2 country code. Default: "US"
- `device` (str): "desktop" or "mobile". Default: "desktop"
- `pages` (int): Number of pages to scrape (1-10). Default: 1
- `include_aioverview` (bool): Include Google AI Overview. Default: False
- `aioverview_markdown` (bool): Format AI Overview as markdown. Default: False
- `include_html` (bool): Include raw HTML response. Default: False

### CloroChatGPT

Extract structured data from ChatGPT with shopping cards, entity extraction, and advanced features for monitoring products, prices, and brand mentions.

```python
from langchain_cloro import CloroChatGPT

tool = CloroChatGPT()

# Basic query
results = tool.invoke({"prompt": "What are the best sneakers under $100?"})

# With shopping card data
results = tool.invoke({
    "prompt": "best running shoes",
    "include_raw_response": True,
    "include_search_queries": True
})
```

**Parameters:**
- `prompt` (str, required): The prompt/query
- `country` (str): ISO 3166-1 alpha-2 country code. Default: "US"
- `include_raw_response` (bool): Include raw streaming response events. Default: False
- `include_search_queries` (bool): Include search fan-out queries. Default: False
- `include_html` (bool): Include HTML response. Default: False
- `include_markdown` (bool): Include markdown response. Default: False

### CloroGemini

Extract structured data from Google's Gemini AI with source citations and multiple output formats.

```python
from langchain_cloro import CloroGemini

tool = CloroGemini()

results = tool.invoke({"prompt": "Explain quantum entanglement"})
```

**Parameters:**
- `prompt` (str, required): The prompt/query
- `country` (str): ISO 3166-1 alpha-2 country code. Default: "US"
- `include_html` (bool): Include HTML response. Default: False
- `include_markdown` (bool): Include markdown response. Default: False

### CloroBingAIMode

Get the answer that Bing AI Mode gives to a prompt, with its sources and inline citation pills.

```python
from langchain_cloro import CloroBingAIMode

tool = CloroBingAIMode()

results = tool.invoke({"prompt": "What are the best project management tools?"})

# From a US state, with the answer as Markdown
results = tool.invoke({
    "prompt": "best pizza near me",
    "country": "US",
    "state": "NY",
    "include_markdown": True
})
```

**Parameters:**
- `prompt` (str, required): The prompt/query. Max 3,500 characters after URL encoding (fewer for non-Latin scripts)
- `country` (str): ISO 3166-1 alpha-2 country code. Default: "US"
- `state` (str): Two-letter US state code. Use it only with country "US". Default: None
- `include_html` (bool): Include HTML response. Default: False
- `include_markdown` (bool): Include markdown response. Default: False

The response `result` has `text`, `sources`, `citationPills`, and `markdown` or `html` when you ask for them. A request costs 4 credits, plus 2 for a sync request, plus 2 for a US state.

### get_countries Utility

Get list of supported country codes for specific AI providers.

```python
from langchain_cloro import get_countries

# Get all countries
all_countries = get_countries()

# Get countries for specific model
chatgpt_countries = get_countries(model="chatgpt")
google_countries = get_countries(model="google")
```

## Usage with LangChain Agents

### With an Agent

```python
from langchain.agents import initialize_agent, AgentType
from langchain_openai import OpenAI
from langchain_cloro import CloroGoogleSearch, CloroChatGPT

llm = OpenAI(temperature=0)
tools = [CloroGoogleSearch(), CloroChatGPT()]

agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True,
)

agent.run("What are the latest developments in AI?")
```

### With LCEL

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI
from langchain_cloro import CloroChatGPT

chatgpt = CloroChatGPT()

prompt = ChatPromptTemplate.from_messages([
    ("system", "Answer based on the AI's response:\n\n{response}"),
    ("user", "{question}")
])

chain = {
    "response": lambda x: chatgpt.invoke({"prompt": x["question"]}),
    "question": lambda x: x["question"]
} | prompt | ChatOpenAI() | StrOutputParser()

response = chain.invoke({"question": "What is LangChain?"})
print(response)
```

### Multiple Tools Example

```python
from langchain_cloro import (
    CloroGoogleSearch,
    CloroChatGPT,
    CloroGemini,
    CloroBingAIMode,
)

tools = [
    CloroGoogleSearch(),  # For search queries
    CloroChatGPT(),      # For shopping/product queries
    CloroGemini(),       # For general AI queries
    CloroBingAIMode(),   # For Bing AI answers with citations
]
```

## Supported Country Codes

Use the `get_countries()` utility to fetch supported countries:

```python
from langchain_cloro import get_countries

# Check which countries are available for each model
models = ["google", "chatgpt", "gemini", "bing_aimode"]

for model in models:
    countries = get_countries(model=model)
    print(f"{model}: {len(countries)} countries")
```

## Development

### Running Tests

```bash
# Unit tests
pytest tests/unit_tests -v

# Linting
ruff check .
ruff format .

# Type checking
mypy langchain_cloro
```

### Release

The [Publish workflow](.github/workflows/publish.yml) publishes the package to PyPI when a version tag is pushed. It uses PyPI Trusted Publishing, so you do not need a PyPI token.

1. Increase `version` in `pyproject.toml` and commit the change.
2. Tag the commit with the same version and push the commit and the tag. The workflow runs the unit tests, builds and publishes:

   ```bash
   git tag v0.3.2
   git push origin main v0.3.2
   ```

## API Reference

For detailed API documentation, see [https://cloro.dev/docs](https://cloro.dev/docs/).

## License

MIT

## Links

- Documentation: [https://cloro.dev/docs](https://cloro.dev/docs/)
- Source: [https://github.com/cloro-dev/langchain-cloro](https://github.com/cloro-dev/langchain-cloro)
- cloro API: [https://cloro.dev](https://cloro.dev/)
