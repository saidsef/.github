# HuggingFace smolagents - Build Agents

## Introduction
This documentation provides a detailed guide on setting up and using the [smolagents](https://huggingface.co/docs/smolagents/en/guided_tour) application. The document includes an overview of smolagents, necessary prerequisites for installation, the purpose of the application, a step-by-step guide for setup, and a conclusion for wrapping up.

## What is smolagents?
[smolagents is a software library](https://huggingface.co/docs/smolagents/en/tutorials/building_good_agents) designed to facilitate the creation of AI agents capable of performing specific tasks. It integrates with Hugging Face's models and provides tools to enhance the capabilities of AI agents, making the development of intelligent solutions more accessible.

## Agentic
In the context of smolagents, [agentic](https://blogs.nvidia.com/blog/what-is-agentic-ai/) refers to the app's capability to autonomously handle user prompts, process information, and generate responses using AI models.

AI chatbots currently utilise generative AI to respond to individual queries using natural language processing. The future of AI, however, lies in agentic AI, which features advanced reasoning and planning to independently solve complex, multi-step tasks. This technology promises to improve productivity across various sectors. For example, in customer service, an AI agent could exceed basic question-answering by managing tasks like checking balances and suggesting payment solutions, then proceeding based on user decisions.

## Prerequisite
Before setting up the smolagents application, ensure you have the following prerequisites:
- A working Python environment (Python 3.10+ recommended).
- Streamlit installed in your Python environment.
- An active Hugging Face account and API token for accessing models.
- Basic understanding of AI and natural language processing.

## What does this app do?
The smolagents application demonstrates an AI agent’s capabilities by using Hugging Face's API. Users input a prompt, and the agent leverages AI models to generate detailed responses. The app can be configured to enhance responses by enabling tools like web search and trending model identification.
- Entering prompts and receiving AI-generated responses.
- Enabling/disabling tools such as the Web Search Tool and Trending Model Tool.
- Integrating with different AI models hosted on Hugging Face Hub.

```python
@tool
def model_trending_tool(task: str) -> str:
  """
  This is a tool that returns the most trending model of a given task on the Hugging Face Hub.
  It returns the name of the checkpoint.

  Args:
    task: The task for which to get the trending count.
  """
  most_trending_model = next(iter(list_models(task=task, sort="trending_score", direction=-1)))
  return load_tool(most_trending_model.id, trust_remote_code=True)
```

To load individual tools:
```python
web_search_tool = load_tool("web_search", trust_remote_code=True)
```

## Step-by-Step Guide on Setup

1. **Set Up Your Environment:**
- Ensure Python and libraries (`streamlit`, `smolagents`, `huggingface_hub`) are installed in your system.

2. **Install Required Dependencies:**
- The dependencies are specified in `smolagents/requirements.txt`. Use the following command to install them:
```bash
python -m venv venv
source venv/bin/activate
pip install -r smolagents/requirements.txt
```

3. **Configure Environment Variables:**
- Obtain your Hugging Face API token from https://huggingface.co and set it as an environment variable:
```bash
export HUGGINGFACE_TOKEN='your_huggingface_token'
```

4. **Agent Interaction:**
- Navigate to the directory containing the app file `smolagents/app.py`.

```python
agent = CodeAgent(
  tools=options["tools"],
  model=options["model"],
  additional_authorized_imports=options["additional_authorized_imports"],
  max_steps=options["max_steps"],
  planning_interval=options["planning_interval"]
)
```
- Execute the app using Streamlit with the following command:
```bash
streamlit run smolagents/app.py
```

5. **Interact with the App:**
- Browser will display the Streamlit interface.
- Enter a prompt in the "Prompt" column and observe the AI agent's response in the "Response" column.
- **Optional:** Enable additional functionalities such as trending tools or web search for enhanced responses.
- The application will process your prompt and display the results in real-time.

## Conclusion
smolagents provides an interactive and flexible platform for exploring AI capabilities with tools from Hugging Face. With a straightforward setup process and an intuitive interface, it allows users to experiment with AI agents effectively. For any issues or contributions, refer to the repository's documentation and support guidelines.
