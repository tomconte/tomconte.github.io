---
title: "Secure Code Execution with LLMs: Building CodeBox-AI with Model Context Protocol (MCP)"
layout: post
---

In recent projects, I've been exploring the potential of large language models (LLMs) not just to generate code snippets but also to execute them safely in isolated environments. This combination creates powerful workflows where AI assistants can iteratively develop, test, and refine solutions while providing visualizations and insights from data analysis. The project was of the "AI augmentation" type, where we take the impressive code generation capabilities of models like GPT-4 and Claude, and extend them with a secure execution environment that lets the AI run its own code, evaluate the results, and refine its approach.

One of the key challenges was building a solution that could execute untrusted code safely, maintain state between executions, and integrate seamlessly with different LLM providers. In this post, I'll introduce CodeBox-AI, an experimental self-hosted Python code execution service, and demonstrate how it leverages the Model Context Protocol (MCP) to provide a standardized interface for LLM-powered applications.

## What is CodeBox-AI?

[CodeBox-AI](https://github.com/tomconte/codebox-ai) is a secure Python code execution service that provides a self-hosted alternative to OpenAI's Code Interpreter or Anthropic's Claude analysis tool. It isolates code execution in Docker containers, supports session-based execution with state persistence, and provides robust security controls to prevent dangerous operations.

The main features include:

- Session-based Python code execution in Docker containers
- IPython kernel for rich output support (including visualizations)
- Dynamic package installation with security controls
- State persistence between executions
- Support for mounting local directories with security controls
- AST-based code analysis to block dangerous operations
- Rather than sending your data to remote services, CodeBox-AI lets you run everything locally, with control over security policies and resource limits.

## The Model Context Protocol (MCP)

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) is an emerging standard that allows LLM applications to interact with tools and resources in a standardized way. MCP enables LLMs to access and manipulate contextual information, execute code, search through documents, and perform various other tasks through a consistent interface.

For CodeBox-AI, MCP provides a standardized way to expose its code execution capabilities to LLM applications. This means the same code execution service can be used with different LLMs without changing the integration code, whether you're using Claude Desktop, a custom GPT, or any other MCP-compatible application.

Let's look at how the MCP implementation works in CodeBox-AI:

```python
@mcp.tool()
async def execute_code(
    code: str, dependencies: Optional[List[str]] = None, ctx: Context = None
) -> List[TextContent | ImageContent]:
    """Execute Python code and return a list of TextContent or ImageContent objects"""
    # Create a session, execute code, and manage the container lifecycle
    # ...
    
    # Parse and return the output
    contents: List[TextContent | ImageContent] = []
    for output in result.get("output", []):
        if output["type"] == "stream" or output["type"] == "result":
            contents.append(types.TextContent(type="text", text=output["content"]))
            
    # Add image outputs
    for file_data in result.get("files", []):
        contents.append(types.ImageContent(type="image", data=file_data, mimeType="image/png"))
        
    return contents
```

This single MCP tool provides the core functionality needed for an LLM to write and execute Python code, with proper error handling and support for rich outputs like images and visualizations.

## Setting up CodeBox-AI

Getting started with CodeBox-AI is straightforward. Here's how you can set it up on your local machine:

1. Clone the repository:

```bash
git clone https://github.com/yourusername/codebox-ai.git
cd codebox-ai
```

2. Install dependencies using `uv` (a faster alternative to `pip`):

```bash
# Install uv if needed
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync
```

3. Running the server:

```bash
# Standard API server
uv run -m codeboxai.main

# MCP server only
uv run -m mcp dev mcp_server.py

# Or combined API+MCP server
uv run run.py
```

The server exposes both a RESTful API for direct integration and an MCP interface for standardized LLM interactions.

## Integrating with LLMs through MCP

One of the most powerful features of CodeBox-AI is its ability to integrate with [Claude Desktop](https://modelcontextprotocol.io/quickstart/user) using the Model Context Protocol. This allows Claude to write and execute code in a controlled environment, visualize results, and refine its approach based on execution feedback.

To register CodeBox-AI with Claude Desktop:

```bash
uv run mcp install mcp_server.py --name "CodeBox-AI"
```

Alternatively, you can configure it manually by editing the Claude Desktop configuration file:

```json
{
  "mcpServers": {
    "CodeBox-AI": {
      "command": "uv",
      "args": [
        "run",
        "--project",
        "/Users/username/src/codebox-ai",
        "/Users/username/src/codebox-ai/mcp_server.py",
        "--mount",
        "/Users/username/Downloads"
      ]
    }
  }
}
```

Once registered, Claude Desktop can use the execution environment provided by CodeBox-AI. This creates a powerful feedback loop where Claude can:

- Generate Python code based on your requirements
- Execute the code in a secure container
- Observe the execution results, including errors
- Refine its approach based on the results
- Persist state between executions (variables, imported libraries, etc.)

## Security Considerations

When letting LLMs execute code, security is a major concern. CodeBox-AI implements several layers of security:

- **Docker isolation**: Each session runs in a separate Docker container, isolated from the host system
- **Code validation**: AST-based analysis to prevent dangerous imports and operations
- **Package controls**: Allowlist/blocklist system for package installation
- **Resource limits**: CPU and memory limits for each container
- **Directory mounting security**: Validation of paths to prevent access to sensitive directories

For example, the code validation looks for dangerous imports:

```python
# AST-based import validation
@ast_rule
def _validate_imports(self, tree: ast.AST) -> Tuple[bool, Optional[str]]:
    """Validates that no dangerous imports are used"""
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for name in node.names:
                base_module = name.name.split(".")[0]
                if base_module in self.forbidden_modules:
                    return False, f"Forbidden import: {name.name}"

        elif isinstance(node, ast.ImportFrom):
            if node.module and node.module.split(".")[0] in self.forbidden_modules:
                return False, f"Forbidden import: {node.module}"
    return True, None
```

The validator blocks dangerous modules like `subprocess`, `socket`, and `pickle`. It also enforces package security by preventing installation of risky packages and enforcing minimum versions for packages with known vulnerabilities.

## Real-world Applications

Let's look at a practical example of using CodeBox-AI with OpenAI's GPT-4 for data analysis:

```python
# Example of using CodeBox-AI with OpenAI for data analysis
messages = [
    {
        "role": "system",
        "content": (
            "You are a helpful AI assistant with the ability to execute Python code. "
            "When a user asks you to perform calculations, create visualizations, or analyze data, you can "
            "write and execute Python code to help them."
        ),
    }
]

# Create a session with mounted directories
session = CodeBoxSession(
    dependencies=["numpy", "pandas", "matplotlib"],
    execution_options={
        "mount_points": [{"host_path": LOCAL_MOUNT_PATH, "container_path": CONTAINER_MOUNT_PATH, "read_only": True}]
    },
)

# Chat with the model and execute code
chat_with_code_execution("Analyze the CSV data in /data/sales.csv and create a visualization of monthly trends", messages, session)
```

This example allows GPT-4 to:

- Access a local CSV file in a mounted directory
- Write Python code to analyze the data
- Create visualizations
- Explain insights from the analysis

All while maintaining security boundaries and presenting rich outputs to the user.

## Conclusion

CodeBox-AI demonstrates the power of combining LLMs with secure code execution environments. By leveraging the emerging Model Context Protocol (MCP), it provides a standardized way for LLMs to write, execute, and iterate on code with proper security controls.

The project is still evolving, but it represents an important step toward more capable AI systems that can not only generate code but also execute, test, and refine it iteratively. This creates a powerful feedback loop where LLMs can learn from execution results and improve their solutions.

For data scientists, developers, and analysts, tools like CodeBox-AI offer a glimpse of a future where AI assistants don't just suggest code but can actively participate in the development and analysis process, accelerating workflows and augmenting human capabilities.

You can find the full source code for CodeBox-AI on GitHub at [https://github.com/tomconte/codebox-ai](https://github.com/tomconte/codebox-ai).
