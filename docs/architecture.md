# Architecture — Microsoft Foundry IQ Knowledge-Enhanced AI Agent

## 1. Architecture Overview

This project implements a Python client that interacts with a Microsoft Foundry agent configured with a Foundry IQ knowledge capability.

The architecture combines:

- A local Python application
- Microsoft Foundry
- A configured Foundry agent
- OpenAI-compatible Conversations and Responses APIs
- Model Context Protocol (MCP) tool interaction
- Human-in-the-loop approval handling
- Foundry IQ
- Azure AI Search
- Enterprise knowledge retrieval

The Python application is responsible primarily for user interaction, conversation management, agent invocation, MCP approval handling, and response presentation.

The knowledge infrastructure and MCP capability are configured separately within Microsoft Foundry and Azure.

---

## 2. High-Level Architecture

```text
+-------------------------+
|          User           |
+------------+------------+
             |
             | Interactive prompt
             v
+-------------------------+
|      Python Client      |
|     agent_client.py     |
+------------+------------+
             |
             | DefaultAzureCredential
             v
+-------------------------+
| Microsoft Foundry       |
| Project                 |
+------------+------------+
             |
             | Agent reference
             v
+-------------------------+
| Product Expert Agent    |
+------------+------------+
             |
             | MCP knowledge capability
             v
+-------------------------+
| Foundry IQ              |
| Knowledge Base          |
+------------+------------+
             |
             v
+-------------------------+
| Azure AI Search         |
+------------+------------+
             |
             v
+-------------------------+
| Enterprise Knowledge    |
+-------------------------+
```

The final response travels back through the Foundry agent to the Python client and is presented to the user.

---

## 3. Architectural Pattern

The application follows a client-to-managed-agent architecture.

The local Python application does not contain the agent's full reasoning configuration, knowledge base, search infrastructure, or MCP server implementation.

Instead, the client connects to an existing Microsoft Foundry project and retrieves a configured agent.

This creates a separation between:

```text
Application Layer
        |
        v
Agent / Orchestration Layer
        |
        v
Knowledge / Tool Layer
        |
        v
Search / Data Layer
```

This separation is important because agent behavior and enterprise knowledge infrastructure can evolve independently from the client application.

---

## 4. Configuration Layer

Application-specific configuration is loaded from environment variables:

```python
load_dotenv()

project_endpoint = os.getenv("PROJECT_ENDPOINT")
agent_name = os.getenv("AGENT_NAME")
```

The application validates both values before attempting to establish a Foundry connection.

```python
if not project_endpoint or not agent_name:
    raise ValueError(
        "PROJECT_ENDPOINT and AGENT_NAME must be set in .env file"
    )
```

The repository provides `.env.example` for configuration guidance while `.env` is excluded from source control.

---

## 5. Identity and Authentication

The client authenticates using Azure Identity:

```python
credential = DefaultAzureCredential(
    exclude_environment_credential=True,
    exclude_managed_identity_credential=True
)
```

The credential is supplied to `AIProjectClient`:

```python
project_client = AIProjectClient(
    credential=credential,
    endpoint=project_endpoint
)
```

This design avoids embedding Azure authentication secrets directly in application source code.

For local development, `DefaultAzureCredential` can resolve an available supported developer identity according to its configured credential chain.

The application itself does not store an Azure password, access token, or Search API key.

---

## 6. Microsoft Foundry Project Client

`AIProjectClient` provides the application-level connection to the configured Microsoft Foundry project.

```python
project_client = AIProjectClient(
    credential=credential,
    endpoint=project_endpoint
)
```

The client is then used for two important operations:

1. Obtaining an OpenAI-compatible client.
2. Retrieving the configured Foundry agent.

```python
openai_client = project_client.get_openai_client()

agent = project_client.agents.get(
    agent_name=agent_name
)
```

This means the Python client references an existing agent instead of defining and provisioning the complete agent configuration locally.

---

## 7. Agent Reference

The application retrieves the configured agent by name:

```python
agent = project_client.agents.get(
    agent_name=agent_name
)
```

When creating a response, the application passes an `agent_reference`:

```python
extra_body={
    "agent_reference": {
        "name": agent.name,
        "type": "agent_reference"
    }
}
```

This allows Microsoft Foundry to execute the server-side agent configuration associated with that agent.

The client therefore does not need to reproduce the agent's instructions, knowledge configuration, or MCP configuration in Python.

---

## 8. Conversation Lifecycle

A new conversation is created when the application starts:

```python
conversation = openai_client.conversations.create(
    items=[]
)
```

The conversation identifier is retained for the active application session.

Each user message is added to the existing conversation:

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

The same conversation is then referenced when the agent is invoked.

This allows subsequent interactions to occur within the same server-side conversation context.

---

## 9. Responses API

After adding the user's message, the application invokes the Foundry agent through the Responses API:

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

The request references:

- The active conversation
- The configured Foundry agent

The resulting response may contain normal response output or MCP approval requests that must be handled before execution can continue.

---

## 10. Foundry IQ Knowledge Layer

The Foundry agent is configured separately with a Foundry IQ knowledge base.

Foundry IQ provides the knowledge-grounding layer used by the agent.

Conceptually:

```text
User Question
     |
     v
Foundry Agent
     |
     v
Foundry IQ
     |
     v
Azure AI Search
     |
     v
Relevant Enterprise Knowledge
     |
     v
Agent Response
```

The Python application does not directly query Azure AI Search.

Instead, knowledge retrieval occurs through the agent's configured knowledge capability.

This is an important architectural boundary.

---

## 11. Azure AI Search Layer

Azure AI Search provides the search infrastructure behind the configured knowledge experience.

Its responsibility is separate from the Python client.

The application does not contain:

- Search administration keys
- Index provisioning logic
- Search service provisioning
- Index creation logic
- Direct Search SDK queries

The Search resource and its connection to the Foundry knowledge environment are managed externally.

---

## 12. MCP Integration Boundary

The project demonstrates MCP-enabled agent interaction, but the Python application does not implement or host the MCP server.

The architectural boundary is:

```text
Python Client
     |
     v
Microsoft Foundry Agent
     |
     v
Configured MCP Knowledge Capability
     |
     v
Foundry IQ / Knowledge Infrastructure
```

The client becomes involved when the agent response contains an MCP approval request.

This distinction prevents the portfolio from incorrectly presenting `agent_client.py` as an MCP server implementation.

---

## 13. MCP Approval Detection

The application inspects response output for items whose type is:

```text
mcp_approval_request
```

The implementation is:

```python
approval_requests = [
    item for item in (getattr(response, "output", None) or [])
    if getattr(item, "type", None) == "mcp_approval_request"
]
```

The use of a list allows the application to support:

- No approval requests
- One approval request
- Multiple approval requests

If no approval requests are present, the approval loop terminates and response processing continues.

---

## 14. Human-in-the-Loop Authorization

When approval is required, the application exposes information about the requested operation before asking the user to make a decision.

The client displays:

```text
Tool Name
MCP Server Label
Tool Arguments
```

The approval prompt accepts:

```text
yes
y
```

as approval.

Other input results in denial.

Conceptually:

```text
MCP Tool Request
      |
      v
Approval Required
      |
      v
Display Tool Details
      |
      v
Human Decision
    /       \
Approve     Deny
    \       /
     v     v
Approval Response
```

This demonstrates a human-in-the-loop governance pattern for agent tool execution.

---

## 15. MCP Approval Response

Each decision is converted into an approval response:

```python
approval_items.append({
    "type": "mcp_approval_response",
    "approval_request_id": approval_request.id,
    "approve": approved
})
```

The approval responses are then written back into the active conversation:

```python
openai_client.conversations.items.create(
    conversation_id=conversation.id,
    items=approval_items
)
```

The agent is invoked again:

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

This allows execution to continue after the user has supplied the required authorization decision.

---

## 16. Repeated Approval Handling

The approval logic executes inside a loop:

```text
Response
   |
   v
Approval Requests?
   |
   +---- No ----> Process Final Response
   |
  Yes
   |
   v
Collect Decisions
   |
   v
Submit Approval Responses
   |
   v
Request Next Agent Response
   |
   +----------> Repeat
```

This matters because an agent interaction may potentially encounter additional approval requests after execution continues.

The application does not assume that only one approval cycle can occur.

---

## 17. Client-Side Conversation History

In addition to the server-side conversation, the application maintains a local Python list:

```python
conversation_history = []
```

User messages are recorded as:

```python
{
    "role": "user",
    "content": user_message
}
```

Successful assistant responses are recorded as:

```python
{
    "role": "assistant",
    "content": response_text
}
```

The `history` command displays this local record.

This client-side history should not be confused with the Foundry conversation object.

The two serve different purposes:

| Component | Purpose |
| --- | --- |
| Foundry conversation | Server-side conversation context |
| `conversation_history` | Local application display/history |

---

## 18. Response Processing

Once no additional MCP approval requests remain, the application checks:

```python
if response and response.output_text:
```

The final text is printed to the terminal and added to the local conversation history.

If no response text is available, the application reports:

```text
No response received.
```

---

## 19. Citation Handling

The application includes conditional citation handling:

```python
if hasattr(response, 'citations') and response.citations:
```

If the response object exposes citation information through this property, the application attempts to display it.

The architecture does not assume that citations are always available through a top-level `response.citations` property.

Therefore, citation display should be considered conditional rather than a guaranteed behavior of every agent response.

---

## 20. Interactive Application Loop

The main application loop supports three interaction patterns.

### Normal Message

Any normal input is sent to:

```python
send_message_to_agent(user_input)
```

### History

Entering:

```text
history
```

calls:

```python
display_conversation_history()
```

### Quit

Entering:

```text
quit
```

terminates the session.

Keyboard interruption is also handled.

---

## 21. Exception Handling

Agent operations are wrapped in:

```python
try:
    ...
except Exception as e:
    ...
```

This prevents a single agent/client exception from terminating the entire application without a readable error message.

The main interaction loop also handles:

```python
KeyboardInterrupt
```

and unexpected application exceptions.

For a production implementation, this could be extended with:

- Structured logging
- Correlation IDs
- Retry policies
- Azure Monitor
- Application Insights
- Centralized exception telemetry

---

## 22. Security Boundaries

The architecture separates credentials and infrastructure configuration from application source.

### Repository Boundary

The repository excludes:

```text
.env
.venv/
venv/
env/
__pycache__/
```

### Application Boundary

The Python client does not contain:

- Azure passwords
- Access tokens
- Search API keys
- Embedded cloud credentials

### Tool Authorization Boundary

MCP operations can require explicit human authorization before execution.

### Infrastructure Boundary

The repository does not provision or contain the configuration for:

- Microsoft Foundry resources
- Azure AI Search
- Foundry IQ knowledge bases
- MCP server infrastructure
- Search credentials

---

## 23. Component Responsibility Matrix

| Component | Primary Responsibility |
| --- | --- |
| User | Provides questions and approval decisions |
| Python client | Interaction, conversation calls, approval handling |
| `DefaultAzureCredential` | Azure authentication |
| `AIProjectClient` | Foundry project access |
| OpenAI-compatible client | Conversations and Responses API operations |
| Foundry agent | Agent reasoning and orchestration |
| MCP knowledge capability | Tool-mediated knowledge access |
| Foundry IQ | Knowledge grounding |
| Azure AI Search | Search and retrieval infrastructure |
| Local history list | Client-side conversation display |

---

## 24. Request Execution Flow

A complete interaction follows this sequence:

```text
1. User enters a question
          |
          v
2. Python client adds message to conversation
          |
          v
3. Responses API invokes Foundry agent
          |
          v
4. Agent determines whether knowledge/tool access is needed
          |
          v
5. Configured MCP capability is invoked
          |
          v
6. Approval request may be returned
          |
          v
7. Python client displays requested operation
          |
          v
8. User approves or denies
          |
          v
9. Approval response is added to conversation
          |
          v
10. Agent execution continues
          |
          v
11. Additional approvals are handled if necessary
          |
          v
12. Final response text is returned
          |
          v
13. Python client displays response
          |
          v
14. Response is recorded in local history
```

Not every interaction is guaranteed to require an approval request; the approval branch is processed when such a request is returned.

---

## 25. Implementation Boundaries

This repository demonstrates **application integration**, not full infrastructure provisioning.

Implemented in the repository:

```text
Python interactive client
Azure authentication integration
Foundry project connectivity
Agent lookup
Conversation creation
Conversation message submission
Responses API execution
MCP approval detection
Human approval/denial
Approval response submission
Repeated approval handling
Response presentation
Local conversation history
Exception handling
```

Configured externally:

```text
Microsoft Foundry project
Foundry agent
Agent instructions
Foundry IQ knowledge base
Azure AI Search resources
Knowledge indexes
MCP knowledge capability
Search authentication
Azure RBAC / service permissions
```

This distinction is intentional and should remain explicit when presenting the project.

---

## 26. Relationship to Earlier Foundry IQ Work

This project extends beyond a foundational knowledge-grounding implementation.

The architectural emphasis here is the integration of:

```text
Enterprise Knowledge
        +
Foundry Agent
        +
MCP Knowledge Capability
        +
Programmatic Python Client
        +
Conversation APIs
        +
Human Approval Controls
```

The key engineering progression is therefore not merely using retrieval-augmented generation.

It is integrating enterprise knowledge retrieval into an agent-driven application while managing tool authorization and conversational execution programmatically.

---

## 27. Production Evolution

A production version could extend the architecture with:

```text
                    +----------------------+
                    |   Web / API Layer    |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    | Agent Application    |
                    +----------+-----------+
                               |
              +----------------+----------------+
              |                                 |
              v                                 v
     +------------------+             +------------------+
     | Microsoft        |             | Observability    |
     | Foundry Agent    |             | / Telemetry      |
     +--------+---------+             +------------------+
              |
              v
     +------------------+
     | MCP / Foundry IQ |
     +--------+---------+
              |
              v
     +------------------+
     | Azure AI Search  |
     +------------------+
```

Potential production additions include:

- Azure Key Vault
- Managed identity
- Application Insights
- Azure Monitor
- Structured logging
- Automated evaluation
- Retrieval-quality evaluation
- Role-based approval policies
- Persistent application state
- CI/CD
- Infrastructure as Code
- Automated security scanning
- Automated tests
- Hosted web or API interface

---

## 28. Engineering Competencies Demonstrated

This architecture demonstrates practical experience with:

- Microsoft Foundry
- Foundry agents
- Foundry IQ
- Azure AI Search
- Model Context Protocol
- MCP approval workflows
- Human-in-the-loop agent governance
- Azure Identity
- Azure AI Projects SDK
- OpenAI-compatible APIs
- Conversations API
- Responses API
- Agent references
- Python application architecture
- Environment-based configuration
- Conversation state
- Enterprise knowledge grounding
- Security boundaries
- Exception handling

---

## 29. Summary

The project demonstrates an intermediate enterprise-agent architecture in which a Python application interacts with a managed Microsoft Foundry agent that can access knowledge through Foundry IQ and an MCP-enabled knowledge capability.

The principal architectural separation is:

```text
Python Application
       |
       v
Microsoft Foundry Agent
       |
       v
MCP Knowledge Capability
       |
       v
Foundry IQ
       |
       v
Azure AI Search
       |
       v
Enterprise Knowledge
```

The Python client adds an additional governance layer by detecting MCP approval requests and allowing a human to approve or deny requested operations before agent execution continues.

This combination of **knowledge grounding, agent orchestration, conversational execution, MCP integration, and human authorization** is the primary engineering competency demonstrated by the project.
