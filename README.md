# Hermes Agent

A fork of [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — an autonomous AI agent framework powered by Hermes models with tool-use and function-calling capabilities.

> **Personal fork** — I'm using this to experiment with local LLM agents via Ollama. Main changes from upstream: lower default temperature, increased max iterations.

## Features

- 🤖 **Hermes Model Integration** — Optimized for NousResearch Hermes series models
- 🛠️ **Tool Use & Function Calling** — Structured JSON tool calls following Hermes prompt format
- 🔄 **Agentic Loops** — Multi-step reasoning and action execution
- 🐳 **Docker Support** — Containerized deployment ready
- 🔌 **Extensible Tools** — Easily add custom tools and integrations

## Quick Start

### Prerequisites

- Python 3.10+
- An OpenAI-compatible API endpoint (e.g., [vLLM](https://github.com/vllm-project/vllm), [Ollama](https://ollama.com), or OpenAI)

### Installation

```bash
# Clone the repository
git clone https://github.com/your-org/hermes-agent.git
cd hermes-agent

# Install dependencies
pip install -r requirements.txt

# Copy and configure environment variables
cp .env.example .env
# Edit .env with your API keys and settings
```

### Configuration

Copy `.env.example` to `.env` and configure the following key variables:

```env
# Model / API Configuration
OPENAI_API_BASE=http://localhost:11434/v1
OPENAI_API_KEY=ollama
MODEL_NAME=NousResearch/Hermes-3-Llama-3.1-8B

# Agent Settings
MAX_ITERATIONS=20
TEMPERATURE=0.3
```

> **Note:** The defaults above are set for local Ollama usage. If using vLLM or OpenAI, update `OPENAI_API_BASE` and `OPENAI_API_KEY` accordingly.

### Running the Agent

```bash
python main.py
```

### Docker

```bash
# Build the image
docker build -t hermes-agent .

# Run with environment file
docker run --env-file .env hermes-agent
```

## Usage

```python
from hermes_agent import HermesAgent
from hermes_agent.tools import WebSearchTool, CodeExecutionTool

agent = HermesAgent(
    model="NousResearch/Hermes-3-Llama-3.1-8B",
    tools=[WebSearchTool(), CodeExecutionTool()],
    max_iterations=20,
)

result = agent.run("Research the latest developments in quantum computing and summarize them.")
print(result)
```

## Project Structure

```
hermes-agent/
├── hermes_agent/          # Core agent package
│   ├── __init__.py
│   ├── agent.py           # Main agent loop
│   ├── tools/             # Built-in tools
│   └── utils/             # Utility functions
├── tests/                 # Test suite
├── .env.example           # Environment variable template
├── Dockerfile
├── requirements.txt
└── main.py                # Entry point
```

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feat/my-feature`)
3. Commit your changes following [Conventional Commits](https://www.conventionalcommits.org/)
4. Push and open a Pull Request

Please check existing [issues](../../issues) before opening a new one. Use the provided issue templates.

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgements

- [NousResearch](https://nousresearch.com/)
