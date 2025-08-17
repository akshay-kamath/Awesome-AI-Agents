# Integrating ADK Agents with Model Context Protocol (MCP) Tools

## Overview

This project demonstrates how to integrate Google's Agent Development Kit (ADK) with the **Model Context Protocol (MCP)**, an open standard designed to standardize communication between Large Language Models (LLMs) and external tools, data sources and applications.

The core of this integration is the `MCPToolset` class from the ADK which allows an ADK agent to seamlessly connect to and utilize tools exposed by any MCP-compliant server. This repository explores two primary integration patterns:

1.  **ADK as an MCP Client**: An ADK agent consumes tools from an existing, external MCP server (e.g., Google Maps) to perform tasks like fetching directions.
2.  **Exposing ADK Tools via an MCP Server**: A custom MCP server is built to wrap and expose native ADK tools (like `load_web_page`), making them available to any MCP client.

This project provides hands on examples for both patterns, showcasing the power and flexibility of combining ADK with MCP for building sophisticated, tool-augmented AI agents.

---

## Key Concepts

### What is Model Context Protocol (MCP)?

MCP is an open standard that creates a universal communication layer for LLMs. It operates on a client-server architecture, defining how an MCP server can expose three key resources:
*   **Data (Resources)**: Contextual information the model can query.
*   **Interactive Templates (Prompts)**: Structured inputs for the model.
*   **Actionable Functions (Tools)**: Functions the model can execute.

This standardization allows any MCP-compatible client (like an ADK agent) to discover and use tools from any MCP server without needing custom integration code.

### The `MCPToolset` Class

The `MCPToolset` is the primary mechanism in ADK for integrating with MCP servers. When added to an agent's tool list, it automatically handles all the complexities of the MCP connection:

*   **Connection Management**: Establishes and manages the connection to the MCP server, whether it's a local script or a remote service.
*   **Tool Discovery**: Queries the MCP server for its list of available tools (`list_tools`).
*   **Tool Adaptation**: Converts the discovered MCP tool schemas into ADK-compatible `BaseTool` instances, making them natively available to the agent.
*   **Proxying Calls**: When the agent decides to use a tool, `MCPToolset` transparently proxies the function call (`call_tool`) to the MCP server, handles the arguments, and returns the result to the agent.

---

## Project Structure

```
.
├── adk_mcp_tools/
│   ├── google_maps_mcp_agent/   # Example 1: ADK as an MCP Client
│   │   ├── agent.py
│   │   └── .env
│   ├── adk_mcp_server/          # Example 2: ADK as an MCP Server
│   │   ├── adk_server.py
│   │   ├── agent.py
│   │   └── .env
│   └── requirements.txt
└── README.md
```

---

## Setup and Installation

### Prerequisites
*   Python 3.x
*   A Google Cloud Platform (GCP) project with the Vertex AI, Routes, and Directions APIs enabled.
*   `gcloud` CLI installed and authenticated.
*   `npx` (included with Node.js) for the Google Maps server example.

### Installation Steps

1.  **Clone the repository  and navigate to the root directory.**

2.  **Install the specific version of ADK used in this project:**
    ```bash
    sudo python3 -m pip install google-adk==1.5.0
    ```

3.  **Install additional Python dependencies:**
    ```bash
    python3 -m pip install -r adk_mcp_tools/requirements.txt
    ```

---

## Usage and Examples

### Example 1: ADK as an MCP Client (Google Maps Agent)

This example demonstrates how an ADK agent can use the external Google Maps MCP server to get directions.

#### 1. Configure your API Key

First, you need a Google Maps API key.

1.  Go to the [Google Cloud Console](https://console.cloud.google.com/).
2.  Navigate to **APIs & Services > Credentials**.
3.  Click **+ CREATE CREDENTIALS** and select **API key**.
4.  Copy the generated key.

#### 2. Set up Environment Variables

Create a `.env` file inside the `adk_mcp_tools/google_maps_mcp_agent/` directory with the following content, replacing the placeholder values:

```env
# ~/adk_mcp_tools/google_maps_mcp_agent/.env

GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=your-gcp-project-id
GOOGLE_CLOUD_LOCATION=your-gcp-region
GOOGLE_MAPS_API_KEY="your-actual-api-key"
```

#### 3. Agent Configuration (`agent.py`)

The agent in `google_maps_mcp_agent/agent.py` is configured to use the `MCPToolset`. It starts the Google Maps MCP server as a local background process using `npx` and communicates with it over standard I/O.

```python
# ~/adk_mcp_tools/google_maps_mcp_agent/agent.py

tools=[
    MCPToolset(
        connection_params=StdioConnectionParams(
            server_params=StdioServerParameters(
                command='npx',
                args=[
                    "-y",
                    "@modelcontextprotocol/server-google-maps",
                ],
                env={
                    "GOOGLE_MAPS_API_KEY": google_maps_api_key
                }
            ),
            timeout=15,
        ),
    )
],
```

#### 4. Run the Agent

Launch the ADK web interface from the project root (`adk_mcp_tools`):

```bash
adk web
```

Open the provided `http://localhost:8000` link, select the `google_maps_mcp_agent`, and try the following prompts:
*   `Get directions from GooglePlex to SFO.`
*   `What's the route from Paris, France to Berlin, Germany?`

The agent will use the tools provided by the MCP server (like `maps_directions`) to answer your questions.

### Example 2: Exposing ADK Tools via an MCP Server

This example shows how to wrap the native ADK `load_web_page` tool in a custom MCP server, making it accessible to another ADK agent.

#### 1. The MCP Server (`adk_server.py`)

The `adk_mcp_tools/adk_mcp_server/adk_server.py` script creates a simple MCP server that exposes the `load_web_page` tool. It listens for requests, invokes the ADK tool and returns the result in an MCP-compliant format.

#### 2. The Client Agent (`agent.py`)

The agent in `adk_mcp_server/agent.py` acts as a client. It uses `MCPToolset` to connect to our custom `adk_server.py` script. Note the `tool_filter` which ensures only the `load_web_page` tool is loaded.

*Make sure to update the `PATH_TO_YOUR_MCP_SERVER_SCRIPT` in this file to the absolute path of `adk_server.py`.*

```python
# ~/adk_mcp_tools/adk_mcp_server/agent.py

tools=[
    MCPToolset(
        connection_params=StdioConnectionParams(
            server_params=StdioServerParameters(
                command="python3", # Command to run your MCP server script
                args=[PATH_TO_YOUR_MCP_SERVER_SCRIPT], # Argument is the path to the script
            ),
            timeout=15,
        ),
        tool_filter=['load_web_page'] # Optional: ensure only specific tools are loaded
    )
],
```

#### 3. Run the Server and Agent

You will need two terminal sessions.

*   **In Terminal 1**, start the custom MCP server:
    ```bash
    python3 ~/adk_mcp_tools/adk_mcp_server/adk_server.py
    ```

*   **In Terminal 2**, launch the ADK web interface from the project root:
    ```bash
    cd ~/adk_mcp_tools
    adk web
    ```

Open the web UI, select the `adk_mcp_server` agent, and query it with:
*   `Load the content from https://example.com.`

The agent will connect to your running MCP server, which will execute the `load_web_page` tool and return the website's content.
