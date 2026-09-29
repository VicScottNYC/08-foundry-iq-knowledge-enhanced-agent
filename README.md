# Microsoft Foundry IQ Knowledge-Enhanced AI Agent

A knowledge-enhanced AI agent built with **Microsoft Foundry**, **Foundry IQ**, **Azure AI Search**, and **Model Context Protocol (MCP)**. This project demonstrates enterprise knowledge grounding, programmatic agent interaction, conversation management, and human-in-the-loop approval of MCP tool calls.

## Project Overview

This project implements an interactive Python client for a Microsoft Foundry product-expert agent connected to a Foundry IQ knowledge base backed by Azure AI Search.

The application demonstrates how to:

- Connect securely to a Microsoft Foundry project.
- Retrieve an existing Foundry agent.
- Create and maintain an agent conversation.
- Submit messages through the Conversations API.
- Execute the agent through the Responses API.
- Integrate an agent with enterprise knowledge through Foundry IQ.
- Detect MCP tool calls requiring approval.
- Inspect tool-call arguments before execution.
- Approve or deny MCP operations interactively.
- Return approval decisions to the agent.
- Continue execution after tool approval.
- Maintain client-side conversation history.
- Display final agent responses.

## Architecture

```text
User
  |
  v
Python Client
  |
  | DefaultAzureCredential
  v
Microsoft Foundry Project
  |
  v
Product Expert Agent
  |
  | MCP Knowledge Tool
  v
Foundry IQ Knowledge Base
  |
  v
Azure AI Search
  |
  v
Enterprise Knowledge
  |
  v
Grounded Agent Response
```

For MCP operations requiring explicit authorization:

```text
Foundry Agent
     |
     v
MCP Approval Request
     |
     v
Python Client
     |
     +--> Display tool and arguments
     |
     v
Human Approval / Denial
     |
     v
MCP Approval Response
     |
     v
Agent Continues Execution
```

See `docs/architecture.md` for the detailed architecture.

## Key Engineering Concepts

### Microsoft Foundry Agent Integration

The application connects to Microsoft Foundry using `AIProjectClient` and retrieves an existing agent by name.

```python
project_client = AIProjectClient(
    credential=credential,
    endpoint=project_endpoint
)

agent = project_client.agents.get(agent_name=agent_name)
```

An OpenAI-compatible client is obtained from the Foundry project:

```python
openai_client = project_client.get_openai_client()
```

### Azure Authentication

The application uses `DefaultAzureCredential`:

```python
credential = DefaultAzureCredential(
    exclude_environment_credential=True,
    exclude_managed_identity_credential=True
)
```

This enables local developer authentication without embedding Azure credentials in the application source.

### Conversations API

A new conversation is created when the client starts:

```python
conversation = openai_client.conversations.create(items=[])
```

User messages are added to the active conversation:

```python
openai_client.conversations.items.create(
    conversation_id=conversation.id,
    items=[
        {
            "type": "message",
            "role": "user",
            "content": user_message
        }
    ],
)
```

### Responses API

The Foundry agent is invoked using an agent reference:

```python
response = openai_client.responses.create(
    conversation=conversation.id,
    extra_body={
        "agent_reference": {
            "name": agent.name,
            "type": "agent_reference"
        }
    },
    input=""
)
```

This keeps the Python application separate from the server-side agent configuration.

## Foundry IQ Knowledge Integration

The Foundry agent is configured with a **Foundry IQ knowledge base** backed by **Azure AI Search**.

Responsibilities are separated across the architecture:

- **Python client** — user interaction, conversation management, and approval handling.
- **Microsoft Foundry agent** — reasoning and orchestration.
- **MCP knowledge tool** — agent access to the configured knowledge capability.
- **Foundry IQ** — knowledge-grounding layer.
- **Azure AI Search** — enterprise information retrieval.

The Python client does **not** implement or host the MCP server. The MCP knowledge capability is configured separately in the Microsoft Foundry environment.

## Human-in-the-Loop MCP Approval

The client examines agent output for:

```text
mcp_approval_request
```

Approval requests are detected with:

```python
approval_requests = [
    item for item in (getattr(response, "output", None) or [])
    if getattr(item, "type", None) == "mcp_approval_request"
]
```

The implementation supports zero, one, or multiple pending approval requests.

For each approval request, the application can display:

- Tool name
- MCP server label
- Tool-call arguments

The user then approves or denies the requested action.

The decision is returned to the conversation as an:

```text
mcp_approval_response
```

This establishes a human authorization boundary between agent reasoning and MCP tool execution.

## Conversation History

The application maintains lightweight client-side conversation history:

```python
conversation_history = []
```

Enter:

```text
history
```

to display messages recorded during the current application session.

This local history is separate from the server-side Foundry conversation.

## Citation Handling

The application checks whether the response exposes citation information:

```python
if hasattr(response, 'citations') and response.citations:
```

When citation data is available through this response interface, the client attempts to display the associated source information.

The application does not assume citations will always be exposed as a top-level response property.

## Project Structure

```text
08-foundry-iq-knowledge-enhanced-agent/
├── .env.example
├── .gitignore
├── README.md
├── agent_client.py
├── requirements.txt
└── docs/
    └── architecture.md
```

## Technologies

- Python
- Microsoft Foundry
- Microsoft Foundry IQ
- Azure AI Search
- Azure AI Projects SDK
- Azure Identity
- OpenAI Responses API
- OpenAI Conversations API
- Model Context Protocol (MCP)
- python-dotenv

## Prerequisites

You need:

- Python 3
- An Azure subscription
- Access to a Microsoft Foundry project
- A configured Microsoft Foundry agent
- A Foundry IQ knowledge base
- Azure AI Search configured for the knowledge base
- Appropriate Azure authentication

The Azure and Foundry resources are configured separately from this repository.

## Installation

Clone the repository:

```bash
git clone <repository-url>
cd 08-foundry-iq-knowledge-enhanced-agent
```

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it on macOS/Linux:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
python -m pip install -r requirements.txt
```

## Configuration

Create your local environment file:

```bash
cp .env.example .env
```

Configure:

```text
PROJECT_ENDPOINT=<your-microsoft-foundry-project-endpoint>
AGENT_NAME=<your-foundry-agent-name>
```

Never commit `.env` or Azure credentials to source control.

## Authentication

For local development, authenticate with an appropriate credential supported by `DefaultAzureCredential`.

For example, when using Azure CLI authentication:

```bash
az login
```

No passwords, Search API keys, access tokens, or other credentials are embedded in `agent_client.py`.

## Running the Application

Start the client:

```bash
python agent_client.py
```

The application connects to the configured Foundry project and agent, creates a conversation, and presents an interactive prompt.

### Interactive Commands

| Command | Purpose |
| --- | --- |
| `history` | Display locally tracked conversation history |
| `quit` | End the application session |

Other input is submitted to the Foundry agent.

## Security Considerations

The repository excludes:

- `.env`
- Azure credentials
- API keys
- Access tokens
- Local virtual environments
- Python cache files

The MCP approval workflow also demonstrates an important agent-security pattern: external tool execution can be placed behind an explicit human authorization boundary rather than automatically approving every requested operation.

## Error Handling

Agent interaction is wrapped in exception handling so client and service errors can be surfaced without embedding credentials in source code.

The interactive application also handles:

- Empty input
- Keyboard interruption
- Agent/client exceptions

Production implementations should extend this with structured logging, telemetry, retry policies, and centralized observability.

## Engineering Skills Demonstrated

- Microsoft Foundry agent integration
- Foundry IQ
- Azure AI Search
- Enterprise knowledge grounding
- MCP-enabled agent architecture
- Human-in-the-loop tool governance
- Azure SDK authentication
- Azure AI Projects SDK
- OpenAI-compatible Foundry APIs
- Responses API
- Conversations API
- Agent references
- Python application development
- Conversation state management
- Environment-based configuration
- Secure credential separation
- Exception handling

## Portfolio Progression

This project represents an intermediate stage in a broader Microsoft Foundry and Azure AI engineering portfolio.

Earlier projects established foundational Foundry agents, agent workflows, generative AI applications, knowledge grounding, AI safety, Microsoft Agent Framework, and MCP-enabled external capabilities.

Project #08 advances those concepts by combining **agent orchestration, enterprise knowledge retrieval, MCP integration, programmatic conversation management, and human approval controls**.

## Implementation Boundaries

This repository contains the Python client application.

The following components are configured externally and are not provisioned by this repository:

- Microsoft Foundry project
- Foundry prompt agent
- Foundry IQ knowledge base
- Azure AI Search resources
- MCP knowledge-tool configuration
- Search connection credentials and permissions

This project demonstrates application integration with an existing Foundry environment rather than infrastructure provisioning.

## Training Context

The initial implementation was developed while completing Microsoft's learning exercise **Integrate an agent with Foundry IQ**.

This portfolio version focuses on the engineering concepts demonstrated by the completed application and does not redistribute Microsoft credentials or training data.

## Future Enhancements

- Structured logging
- Azure Application Insights
- Automated testing
- Retrieval and response evaluation
- Retry and resiliency policies
- Enhanced citation rendering
- Web-based interface
- Azure-hosted deployment
- Infrastructure as Code
- GitHub Actions CI/CD
- Agent and retrieval observability
- Role-based approval policies
- Azure Key Vault integration

## License

This repository is intended for educational and portfolio demonstration purposes.
