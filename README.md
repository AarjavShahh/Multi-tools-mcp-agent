# MCP Multi-Tool AI Assistant

An interactive AI assistant built with the **Model Context Protocol (MCP)**, **LangChain**, **LangGraph**, and **Groq**. The project connects multiple independent MCP servers to a single conversational agent, allowing the model to call external tools for weather information, web search, general knowledge, and academic research.

The assistant uses a LangGraph workflow to decide when a tool is needed, execute the selected tool, and use the tool result to produce a response. Conversation state is retained through an in-memory checkpointer for the lifetime of the running application.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [MCP Servers and Tools with Sample Queries](#mcp-servers-and-tools-with-sample-queries)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Running the Project](#running-the-project)
- [Example Prompts](#example-prompts)
- [How the Workflow Operates](#how-the-workflow-operates)
- [Conversation Memory](#conversation-memory)
- [LangSmith Tracing](#langsmith-tracing)
- [Current Limitations](#current-limitations)
- [Possible Improvements](#possible-improvements)

---

## Project Overview

This project demonstrates how to build a tool-enabled AI assistant using MCP. Instead of placing every integration inside the main application, each external capability is exposed through a separate MCP server.

The main application (`demo1.py`) connects to these servers with `MultiServerMCPClient`, retrieves their available tools, and binds those tools to a Groq-hosted chat model. LangGraph manages the interaction loop:

1. The user enters a prompt.
2. The language model decides whether a tool is needed.
3. If a tool is requested, LangGraph routes the request to the tool execution node.
4. The selected MCP tool runs and returns its result.
5. The result is passed back to the language model.
6. The assistant generates a final response for the user.

This architecture keeps tool implementations separate from the conversational agent and makes it easier to add more MCP servers in the future.

## Features

- **Multiple MCP integrations:** Connects five MCP servers from one application.
- **Weather lookup:** Retrieves current weather information for a specified location.
- **Web search:** Uses Tavily and DuckDuckGo to retrieve relevant internet search results.
- **Wikipedia search:** Retrieves concise information about general topics, people, places, and events.
- **Academic research search:** Searches arXiv for scientific and research papers.
- **LLM-based tool selection:** The model selects tools based on the user's request.
- **LangGraph orchestration:** Uses a state graph and conditional routing to manage tool calls.
- **Conversation state:** Uses LangGraph's in-memory `MemorySaver` checkpointer.
- **Two MCP transports:** Demonstrates both `stdio` and `streamable-http` transports.
- **Optional observability:** Includes LangSmith tracing configuration.

## Architecture

                 +------------------+
                 |       USER       |
                 +------------------+
                          |
                          v
                 +------------------+
                 |  Main Application|
                 +------------------+
                          |
                          v
                 +------------------+
                 | MultiServerMCP   |
                 |     Client       |
                 +------------------+
                          |
         +----------------+----------------+
         |        |       |       |         |
         v        v       v       v         v
      Weather  Tavily  Wikipedia arXiv  DuckDuckGo
        MCP     MCP      MCP      MCP      MCP
         |        |       |       |         |
         +--------+-------+-------+---------+
                          |
                          v
                 +------------------+
                 |   LangGraph      |
                 |   StateGraph     |
                 +------------------+
                          |
                          v
                 +------------------+
                 |    Groq LLM      |
                 +------------------+
                          |
                          v
                 +------------------+
                 | Final Response   |
                 +------------------+

### Transport configuration

The project uses two MCP transport types:

| MCP server | Transport | Connection |
|---|---|---|
| Weather | Streamable HTTP | `http://127.0.0.1:8003/mcp` |
| Tavily | Streamable HTTP | `http://127.0.0.1:8004/mcp` |
| Wikipedia | STDIO | Started as a subprocess by the client |
| arXiv | STDIO | Started as a subprocess by the client |
| DuckDuckGo | STDIO | Started as a subprocess by the client |

**Why use two transports?**

- **STDIO:** The main client launches the server process and communicates with it through standard input and output. No separate server terminal is required for these integrations.
- **Streamable HTTP:** The server runs as an HTTP service. The main client connects to its MCP endpoint, so the HTTP server must be started separately.

## Project Structure

```text
MCP_PROJECT_3/
│
├── demo1.py                 # Main interactive AI assistant and LangGraph workflow
│
├── weather_server.py        # MCP server for current weather information
├── tavily_server.py         # MCP server for Tavily web search
├── wikipedia_server.py      # MCP server for Wikipedia search
├── arxiv_server.py          # MCP server for arXiv research-paper search
├── duckduckgo_server.py     # MCP server for DuckDuckGo search
│
├── src/
│   └── mcp_project_3/
│       └── __init__.py      # Package entry point placeholder
│
├── .env                     # Local environment variables (do not commit)
├── .python-version          # Python version selection
├── pyproject.toml           # Project metadata and dependencies
├── requirements.txt         # Dependency file (currently empty in this project)
├── uv.lock                  # Locked dependency versions for uv
└── README.md
```

> The runnable interactive client in this repository is `demo1.py`. The package entry point in `src/mcp_project_3/__init__.py` is currently only a placeholder and is not the application launcher.

## MCP Servers and Tools with Sample Queries

Each server exposes a Python function as an MCP tool using the `@mcp.tool()` decorator from FastMCP.

### 1. Weather Server

**File:** `weather_server.py`  
**MCP server name:** `weather`  
**Transport:** Streamable HTTP  
**Port:** `8003`  
**Tool:** `get_weather(location: str)`

Uses LangChain's `OpenWeatherMapQueryRun` and `OpenWeatherMapAPIWrapper` to retrieve current weather information for a location.

Example input:

```text
Surat, India
```

Requires an OpenWeatherMap API key.

### 2. Tavily Search Server

**File:** `tavily_server.py`  
**MCP server name:** `tavily`  
**Transport:** Streamable HTTP  
**Port:** `8004`  
**Tool:** `search_web(query: str)`

Uses LangChain's `TavilySearchResults` with `max_results=5` to search the web and return relevant results.

Useful for current information and web research. Requires a Tavily API key.

### 3. Wikipedia Server

**File:** `wikipedia_server.py`  
**MCP server name:** `wiki`  
**Transport:** STDIO  
**Tool:** `search_wikipedia(query: str)`

Uses `WikipediaQueryRun` and `WikipediaAPIWrapper` to search Wikipedia.

Configuration in the project:
- Maximum results: `2`
- Maximum document content length: `500` characters

Useful for general background information and introductory explanations.

### 4. arXiv Server

**File:** `arxiv_server.py`  
**MCP server name:** `arxiv`  
**Transport:** STDIO  
**Tool:** `search_arxiv(query: str)`

Uses `ArxivQueryRun` and `ArxivAPIWrapper` to search academic papers hosted on arXiv.

Configuration in the project:
- Maximum results: `2`
- Maximum document content length: `1000` characters

Useful for finding research related to artificial intelligence, machine learning, deep learning, and other scientific subjects.

### 5. DuckDuckGo Server

**File:** `duckduckgo_server.py`  
**MCP server name:** `duckduckgo`  
**Transport:** STDIO  
**Tool:** `search_duckduckgo(query: str)`

Uses `DuckDuckGoSearchRun` and `DuckDuckGoSearchAPIWrapper` to search the internet through DuckDuckGo.

This integration does not require a user-provided API key in the current implementation, although availability may depend on the search service and network conditions.

## Technology Stack

| Technology | Purpose |
|---|---|
| Python 3.12+ | Main programming language |
| Model Context Protocol (MCP) | Standardized communication between the assistant and tools |
| FastMCP | Creating MCP servers and exposing Python functions as tools |
| LangChain | Tool integrations and LLM integration |
| LangChain MCP Adapters | Connecting the main application to multiple MCP servers |
| LangGraph | Building and running the agent workflow |
| Groq | Hosting and serving the chat model |
| Qwen model | Language model configured in the application |
| OpenWeatherMap | Weather information |
| Tavily | Web search |
| Wikipedia | General knowledge retrieval |
| arXiv | Academic research-paper search |
| DuckDuckGo | Web search |
| python-dotenv | Loading environment variables from `.env` |
| LangSmith | Optional LLM/application tracing |

## Prerequisites

Before running the project, make sure you have:

- Python **3.12 or later**
- Git
- A code editor such as Visual Studio Code
- Internet connectivity
- A Groq API key
- An OpenWeatherMap API key
- A Tavily API key
- A LangSmith API key if you want to use the tracing configured by the current client code

The project uses the `uv.lock` file, so installing dependencies with `uv` is recommended.

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/<your-repository-name>.git
```

Move into the project directory:

```bash
cd MCP_PROJECT_3
```

Replace the repository URL above with your actual GitHub repository URL.

### 2. Create a virtual environment

Using uv python package. UV is extremely fast python package which is written in RUST. Aalternative to tools such as pip, venv, virtualenv, pip-tools etc.

**Windows PowerShell**

```powershell
uv venv
.venv\Scripts\activate
```

**Windows Command Prompt**

```cmd
uv venv
.venv\Scripts\activate
```


If PowerShell blocks activation, you may need to adjust the execution policy for your current user or use Command Prompt instead.

### 3. Install dependencies

**Recommended: using uv**

Install `uv` if it is not already installed:

```bash
pip install uv
```

Then, from the project root, install the dependencies recorded in `pyproject.toml` and `uv.lock`:

```bash
uv sync
```

If you prefer to install into the already activated virtual environment using pip:

```bash
pip install -e .
```

> `requirements.txt` is currently empty in this repository. Use `uv sync` or `pip install -e .` rather than `pip install -r requirements.txt`.

### 4. Configure environment variables

Create a `.env` file in the project root (the same directory as `demo1.py`).

Add the following variables:

```env
# Groq LLM
GROQ_API_KEY=your_groq_api_key

# OpenWeatherMap
OPENWEATHERMAP_API_KEY=your_openweathermap_api_key

# Tavily Search
TAVILY_API_KEY=your_tavily_api_key

# LangSmith tracing
LANGSMITH_API_KEY=your_langsmith_api_key
```

Get API keys from the providers:

- Groq: https://console.groq.com/keys
- OpenWeatherMap: https://openweathermap.org/api
- Tavily: https://tavily.com/
- LangSmith: https://smith.langchain.com/

**Important:** The current `demo1.py` checks for both `GROQ_API_KEY` and `LANGSMITH_API_KEY`. It also enables LangSmith tracing and sets the project name to `MCP_PROJECT_3`.

The weather and Tavily servers load the `.env` file from their working directory. Run the commands from the project root so they can find the same `.env` file.

## Running the Project

The main client connects to the Weather and Tavily servers through HTTP. Start these two servers before launching the client.

You will need **three terminals** open in the project root.

### Terminal 1: Start the Weather MCP server

Activate the virtual environment, if it is not already active, and run:

```bash
python weather_server.py
```

The server is configured to listen on:

```text
http://127.0.0.1:8003
```

### Terminal 2: Start the Tavily MCP server

Activate the virtual environment, if it is not already active, and run:

```bash
python tavily_server.py
```

The server is configured to listen on:

```text
http://127.0.0.1:8004
```

### Terminal 3: Start the main AI assistant

Run:

```bash
python demo1.py
```

The application will:

1. Initialize the MCP client connections.
2. Start the STDIO MCP servers (Wikipedia, arXiv, and DuckDuckGo) as subprocesses.
3. Retrieve the available tools from all configured servers.
4. Initialize the configured Groq chat model.
5. Bind the MCP tools to the model.
6. Build and compile the LangGraph workflow.
7. Start the interactive command-line chat.

You should see the available tool names printed in the terminal. The exact output depends on successful server connections.

### Interact with the assistant

Enter a prompt when the application displays:

```text
Enter your prompt:
```

The assistant will process the request and call an appropriate tool when needed.

To stop the application, enter:

```text
exit
```

You can also use:

```text
quit
bye
```

## Example Prompts

Try the following prompts after all required servers have started.

| Use case | Example prompt |
|---|---|
| Weather | `What is the current weather in Surat, India?` |
| Weather | `Check the weather in Mumbai, India.` |
| Tavily web search | `Search the web for the latest developments in generative AI.` |
| Tavily web search | `Find recent information about the Model Context Protocol.` |
| Wikipedia | `Explain the history of Python programming language.` |
| Wikipedia | `Who was Alan Turing?` |
| arXiv research | `Find research papers about large language models.` |
| arXiv research | `Search arXiv for machine learning applications in healthcare.` |
| DuckDuckGo | `Find useful resources for learning LangGraph.` |
| DuckDuckGo | `Search for beginner-friendly tutorials on MCP servers.` |
| Multi-step research | `Research the applications of AI agents and summarize the key findings.` |

Tool selection is performed by the language model. If a request is ambiguous, specify the source you want (for example, “search arXiv” or “use Wikipedia”).

## How the Workflow Operates

The main application is implemented in `demo1.py`.

### 1. Load environment variables

The application uses `python-dotenv` to load the `.env` file relative to the project directory.

### 2. Connect to MCP servers

`MultiServerMCPClient` is configured with five servers:

- Weather and Tavily use Streamable HTTP.
- Wikipedia, arXiv, and DuckDuckGo use STDIO.

### 3. Retrieve tools

The client calls:

```python
tools = await client.get_tools()
```

This retrieves the tools exposed by the connected MCP servers.

### 4. Initialize the language model

The project configures `ChatGroq` with:

```python
model = ChatGroq(
    model="qwen/qwen3.8-27b",
    temperature=0
)
```

The model is configured with a temperature of `0` to favor more consistent responses. Model availability depends on the Groq account and provider support.

### 5. Bind tools to the model

```python
llm_with_tools = model.bind_tools(tools)
```

This makes the discovered MCP tools available to the model for tool calling.

### 6. Define the graph state

The application defines a state containing a message history:

```python
class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
```

The `add_messages` reducer appends new messages to the existing conversation state.

### 7. Build the LangGraph workflow

The graph contains two main nodes:

- **LLM node:** Invokes the language model.
- **Tools node:** Executes requested tools through LangGraph's `ToolNode`.

The conditional routing function `tools_condition` checks whether the model response contains tool calls.

       ┌───────────────┐
       │     START     │
       └───────┬───────┘
               │
               ▼
       ┌───────────────┐
       │    LLM Node   │◄──────────────┐
       └───────┬───────┘               │
               │                       │
               ▼                       │
       ┌───────────────┐               │
       │tools_condition│               │
       └───────┬───────┘               │
               │                       │
          ┌────┴────┐                  │
          │         │                  │
          ▼         ▼                  │
    Tool call   No tool call           │
          │         │                  │
          ▼         ▼                  │
    ┌─────────┐  ┌─────────┐           │
    │ ToolNode│  │   END   │           │
    └────┬────┘  └─────────┘           │
         │                             │
         └─────────────────────────────┘

This creates an agent loop: the model can request a tool, receive the tool result, and then decide whether another tool call is necessary before responding.

### 8. Compile and invoke the graph

The graph is compiled with a `MemorySaver` checkpointer. The application uses a fixed thread ID (`1`) for the interactive session and invokes the graph for each user message.

## Conversation Memory

The project uses LangGraph's in-memory `MemorySaver` checkpointer.

This allows the assistant to retain the message history associated with the configured thread while the application is running.

**Important limitations:**

- Memory is stored in process memory.
- It is not a persistent database.
- Conversation history is lost when the application process stops.
- The current implementation uses a fixed thread ID, so it is designed for one interactive conversation rather than multiple isolated users.

## LangSmith Tracing

The main application enables LangSmith tracing:

```python
os.environ["LANGSMITH_TRACING"] = "true"
os.environ["LANGSMITH_PROJECT"] = "MCP_PROJECT_3"
```

It also checks that `LANGSMITH_API_KEY` is available before starting.

When configured correctly, LangSmith can help inspect application traces, model calls, and tool execution flow.

If you do not want tracing, the current client code must be modified: remove or make optional the `LANGSMITH_API_KEY` validation and tracing configuration. Merely leaving the key blank will currently cause the application to raise an error.


## Current Limitations

The following points describe the current implementation:

- The assistant runs in an interactive terminal; no web UI is included.
- Weather and Tavily require separate HTTP server processes.
- Conversation memory uses an in-memory checkpointer and is not persistent across restarts.
- The client uses a fixed conversation thread ID.
- The main client currently requires a LangSmith API key because of its startup validation.
- The package entry point is a placeholder; run the application with `python demo1.py`.
- `requirements.txt` is empty; dependency metadata is maintained in `pyproject.toml` and `uv.lock`.
- Tool output quality and availability depend on the external services and APIs.

## Possible Improvements

Ideas for extending the project:

- Add more MCP servers and tools (for example, database, file-system, calculator, or code-execution tools).
- Add support for multiple LLM providers and configurable model selection.
- Make LangSmith tracing optional through an environment variable.
- Add persistent conversation storage using a database-backed checkpointer.
- Support multiple users and independent conversation thread IDs.
- Add structured logging and more detailed error handling.
- Add automated tests for individual MCP servers and the agent workflow.
- Add a Streamlit, FastAPI, or web-based chat interface.
- Add Docker and Docker Compose support to simplify multi-server startup.
- Add tool permissions and input validation.

---
