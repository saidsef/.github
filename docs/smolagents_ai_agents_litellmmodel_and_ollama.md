# Building AI Agents with Smolagents, LiteLLMModel, and Ollama

## Preface

This document is a follow-up to previous documentation on [building AI agent using smolagents](https://github.com/saidsef/.github/blob/main/docs/huggingface_smolagents_build_agents.md)

## Introduction

Smolagents, combined with LiteLLMModel and Ollama, provides a brilliant framework for building AI agents that can leverage local Large Language Models (LLMs), it allows you to bypass expensive API calls, drastically reduce your expenditure on API fees. This guide will demonstrate how to create an agent that utilises Ollama's models through LiteLLM's unified interface.

## Understanding the Agentic Architecture

The integration works through:
- Smolagents as the agent framework
- LiteLLMModel as the model interface layer
- Ollama for running local LLMs
- Custom tools for extended capabilities

## Prerequisites

- Ollama installed and running locally
- Python environment with smolagents
- Basic understanding of agent architectures
- Qwen 14B model pulled in Ollama (ollama pull qwen2.5:14b)

## Step-by-Step Guide

1. Basic Agent Setup

First, let's create a basic agent with the required imports:

```python
from smolagents import CodeAgent, LiteLLMModel, tool, load_tool
from huggingface_hub import login, list_models
import logging

logger = logging.getLogger(__name__)
```

2. Configure LiteLLM with Ollama

Set up the LiteLLM model configuration:

```python
model_config = {
    "model_id": "ollama_chat/qwen2.5:14b",
    "api_base": "http://localhost:11434",
    "api_key": "ollama"  # Required but not utilised by Ollama
}

llm = LiteLLMModel(**model_config)
```

3. Create Custom Tools

Add tools to enhance your agent's capabilities:

```python
@tool
def web_search_tool(query: str) -> str:
    """
    Performs a web search using the provided query.
    Args:
        query: Search query string
    """
    web_search = load_tool("web_search", trust_remote_code=True)
    return web_search(query)

@tool
def model_trending_tool(task: str) -> str:
    """
    Finds trending models for a specific task.
    Args:
        task: The task category to search
    """
    most_trending_model = next(iter(list_models(
        task=task,
        sort="trending_score",
        direction=-1
    )))
    return most_trending_model.id
```

4. Initialise the Agent

Combine everything into a functional agent:

```python
def create_agent(additional_tools=None):
    options = {
        "additional_authorized_imports": [
            "requests", "bs4", "markdownify",
            "numpy", "pandas", "matplotlib"
        ],
        "max_steps": 5,
        "planning_interval": 3,
        "tools": additional_tools or []
    }

    agent = CodeAgent(
        model=llm,
        tools=options["tools"],
        additional_authorized_imports=options["additional_authorized_imports"],
        max_steps=options["max_steps"],
        planning_interval=options["planning_interval"]
    )

    return agent
```

## Best Practises

- Always initialise Ollama before running the agent
- Monitor memory utilisation when running large models
- Implement error handling for tool execution
- Cache frequent operations for better performance

## Conclusion

This setup provides a robust foundation for building AI agents that run locally whilst maintaining the flexibility to add cloud services when needed. The combination of Smolagents, LiteLLM, and Ollama offers a sophisticated platform for developing intelligent applications.

To execute the example, ensure Ollama is running (ollama serve) and your Python environment has all required dependencies installed.
