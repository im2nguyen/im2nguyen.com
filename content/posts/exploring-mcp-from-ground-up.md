---
title: "Exploring Model Context Protocol (MCP) from the ground up"
date: 2025-03-16T11:31:36-08:00
draft: false
---

Large language models (LLMs) have changed how we interact with artificial intelligence, enabling natural conversations and complex reasoning. Yet these powerful models have a fundamental limitation: they're constrained by the context available during each interaction. Without access to real-time information, private data sources, or the ability to take actions, LLMs remain sophisticated but isolated systems.

Model context protocol (MCP) tackles this limitation by creating standardized interfaces between LLMs and external systems. Similar to how APIs transformed web development by connecting different services, MCPs create clear pathways for language models to access tools, retrieve information, and perform actions based on user requests. This standardization does more than just add convenience. By separating model providers from tool creators, MCPs foster an ecosystem where specialized capabilities can be developed independently yet work together seamlessly. This approach solves the fragmentation problem in AI tool integration, where each model provider currently uses unique and incompatible connection methods.

This article explores both the conceptual foundations and practical implementation of MCPs, organized in three main parts:

1. **What problems do MCPs solve?** examines the practical limitations of LLMs and how MCPs address these challenges, with examples of current production implementations across various industries.

2. **How does MCP work?** explores how hosts, clients, and servers work together to create flexible systems for context enrichment and tool integration, showing the building blocks of effective MCP implementations.

3. **How to implement MCP servers?** provides practical guidance on transport mechanisms, security considerations, and design patterns that help build robust, production-ready MCP systems.

Throughout this journey, each section builds on previous knowledge, showing how MCPs transform isolated language models into integrated components of sophisticated AI applications. 

## MCP CLI and `mcp-servers`

To help make MCP concepts more concrete, I created two open-source tools: the MCP CLI ([`mcp-cli`](https://github.com/im2nguyen/mcp-cli)) and several MCP server implementations ([`mcp-servers`](https://github.com/im2nguyen/mcp-servers)). These MIT-licensed resources offer a hands-on way to see the concepts we've discussed in action.

Using these tools is completely optional. The MCP concepts stand on their own, but seeing them in action helps bridge the gap between theory and implementation. They illustrate how clients connect to servers, how capabilities are discovered, and how the protocol's message format works in practice.

If you'd like to try these tools yourself, follow these steps:

First, install the MCP CLI.

```bash
npm install -g @im2nguyen/mcp
```

Then, clone the `mcp-servers` repository. These servers provide different functionality such as calculator, weather, and file management capabilities. The examples in this article use the calculator and weather servers.

```bash
git clone https://github.com/im2nguyen/mcp-servers.git
```

Create a file named `mcp-config.json` in your working directory. You'll need to update the paths to point to where you cloned the repository:

```json
{
  "mcpServers": {
    "calculator": {
      "command": "node",
      "args": ["path/to/mcp-servers/mcp-calculator-server/index.js"],
      "type": "stdio"
    },
    "calculator-sse": {
      "command": "node",
      "args": ["path/to/mcp-servers/mcp-calculator-sse-server/index.js"],
      "type": "sse",
      "url": "http://localhost:3000"
    },
    "weather": {
      "command": "node",
      "args": ["path/to/mcp-servers/mcp-weather-server/build/index.js"],
      "type": "stdio"
    },
    "file-manager": {
      "command": "node",
      "args": [
        "path/to/mcp-servers/mcp-file-manager-server/build/index.js"
      ],
      "type": "stdio"
    }
  }
}
```

Make sure to replace `path/to/mcp-servers` with the actual path where you cloned the repository.

Import the configuration.

```bash
mcp config import mcp-config.json
```

Ask a simple question using a calculator server with debug output. The debug output shows the actual message exchange between client and server, revealing the protocol in action. You'll see the initialization handshake, tool discovery, and the calculator tool being called.

```bash
echo "what is 2+2?" | mcp chat -s calculator
```

Throughout the rest of this article, I'll include code examples and command outputs from these tools to illustrate key concepts. The examples help show how abstract ideas translate into working code.

If you want to explore more deeply, you can:
- Examine the source code to see how components are implemented
- Modify the servers to add your own tools and capabilities 
- Use the MCP CLI as a starting point for building your own host applications

With this practical foundation established, let's examine the core architecture that makes MCPs work. 

## Part I: What problems do MCPs solve?

Now that we understand what Model Context Protocol is, let's explore the specific limitations it addresses. When deploying LLMs in real applications, developers encounter several persistent challenges that prevent these models from reaching their full potential. These aren't just minor inconveniences but fundamental barriers to creating truly useful AI assistants. By identifying these problems clearly, we can better appreciate how MCP's standardized approach offers effective solutions.

The limitations of LLMs become most apparent when we try to use them for practical applications that require up-to-date information or the ability to take actions. These constraints aren't just technical inconveniences—they fundamentally limit what AI systems can accomplish without additional infrastructure. MCP addresses three critical challenges that have historically constrained LLM applications.

1. **Knowledge cutoffs** represent perhaps the most visible limitation. LLMs can't access information beyond their training date, creating a gap between user expectations and model capabilities. When users ask about recent events or personal information, models simply don't have this data. While retrieval-augmented generation (RAG) systems help by finding relevant information to include in prompts, they typically lack the interactive capabilities needed for truly dynamic applications.

   MCP solves this knowledge constraint by providing standardized access to external data sources through a clear protocol. Rather than trying to train models with all possible information, MCP creates pathways for retrieving specific data when needed. This shifts the paradigm from "model as knowledge repository" to "model as reasoning engine with tool access," enabling more flexible and current responses.

2. **Tool integration fragmentation** creates significant development overhead. Each AI platform implements different methods for connecting models to external tools—OpenAI has function calling, Anthropic has tool use, and other providers have their own approaches. This forces developers to create and maintain separate versions of the same functionality for different AI platforms, increasing both development work and maintenance costs.

   MCP addresses this fragmentation through protocol-based standardization built on JSON-RPC 2.0. By defining consistent patterns for message exchange, capability discovery, and error handling, MCP enables capabilities to be defined once but used by any compatible model. The dynamic discovery mechanism allows models to learn available tools through a formal handshake process, eliminating hardcoded assumptions about functionality.

3. **Security concerns** arise when giving language models access to external systems. Direct API access creates risks ranging from exposing sensitive data to potential system exploitation. Without standardized security boundaries, each integration requires custom safety controls, increasing the likelihood of vulnerabilities.

   The MCP architecture creates natural security boundaries by clearly separating responsibilities between components. Hosts manage user interaction and LLM communication, clients maintain server connections and handle message formatting, and servers provide the actual capabilities. Each component operates within defined constraints, creating multiple layers of security and control.

Despite being relatively new, MCP has already been adopted across diverse sectors. Organizations implement MCP servers to provide secure access to file systems, databases, and cloud storage, letting LLMs search documents and retrieve information while maintaining security controls. In development environments, MCP connections to Git repositories and issue trackers help AI assistants understand code and project status throughout the development lifecycle. Web capabilities extend LLMs beyond their training data through search connectors and browser automation, while enterprise integrations connect models to internal knowledge bases, communication platforms like Slack, and business systems like Salesforce.

These real-world applications demonstrate how MCP transforms language models from isolated systems into integrated components of sophisticated AI solutions. By providing standardized access to external capabilities, MCP enables more flexible, accurate, and useful AI applications across diverse domains, from development workflows to enterprise data management and web interactions.

The next section introduces open-source tools that demonstrate these concepts in practice, providing a concrete foundation for understanding how MCP works. 

## Part II: How does MCP work?

The Model Context Protocol creates a structured approach to connecting language models with external capabilities. This architecture provides a flexible yet consistent framework that adapts to diverse implementation needs while maintaining compatibility across different systems. By understanding the core components and how they interact, we can see how MCP achieves its remarkable flexibility without unnecessary complexity.

Before diving into the technical details, let's see what MCP looks like in action. Here's a debug output from the MCP CLI asking a weather-related question, showing all the behind-the-scenes communication:

```bash
❯ echo "what is the weather in los angeles this week?" | mcp chat -s weather -d
2025-03-17T04:48:37.162Z [DEBUG] client:weather: Transport created successfully
2025-03-17T04:48:37.167Z [INFO]  client:weather: Connecting to server...
Calculator MCP server running on stdio
2025-03-17T04:48:37.223Z [INFO]  client:weather: Connected to server!
2025-03-17T04:48:37.223Z [DEBUG] client:weather: Requesting tools list...
2025-03-17T04:48:37.225Z [DEBUG] client:weather: Raw tools response: {"tools":[{"name":"get-alerts","description":"Get weather alerts for a state","inputSchema":{"type":"object","properties":{"state":{"type":"string","minLength":2,"maxLength":2,"description":"Two-letter state code (e.g. CA, NY)"}},"required":["state"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}},{"name":"get-forecast","description":"Get weather forecast for a location","inputSchema":{"type":"object","properties":{"latitude":{"type":"number","minimum":-90,"maximum":90,"description":"Latitude of the location"},"longitude":{"type":"number","minimum":-180,"maximum":180,"description":"Longitude of the location"}},"required":["latitude","longitude"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}}]}
2025-03-17T04:48:37.225Z [INFO]  client:weather: Discovered 2 tools
2025-03-17T04:48:37.225Z [INFO]  client:weather: Available tools: get-alerts, get-forecast
2025-03-17T04:48:37.225Z [DEBUG] host:chat: Starting chat session with server: weather
2025-03-17T04:48:37.225Z [DEBUG] host:chat: Starting non-interactive chat session (piped input)
2025-03-17T04:48:37.226Z [DEBUG] host:chat: Processing piped query:
2025-03-17T04:48:37.226Z [DEBUG] client:weather: Processing query: what is the weather in los angeles this week?
2025-03-17T04:48:37.227Z [DEBUG] client:weather: Available tools:
2025-03-17T04:48:39.817Z [DEBUG] client:weather: Executing tool call: get-forecast
2025-03-17T04:48:39.818Z [DEBUG] client:weather: Calling tool "get-forecast" with args:
2025-03-17T04:48:40.238Z [DEBUG] client:weather: Tool result: Forecast for 34.0522, -118.2437:
...
2025-03-17T04:48:43.300Z [DEBUG] host:chat: Disconnecting client during shutdown
2025-03-17T04:48:43.300Z [INFO]  client:weather: Disconnecting from server
2025-03-17T04:48:43.301Z [INFO]  client:weather: Disconnected from server
2025-03-17T04:48:43.301Z [DEBUG] host:chat: Chat session ended
```

This log captures the complete lifecycle of an MCP interaction:

1. The **host** (MCP CLI) initiates the process by sending a query about LA's weather
2. The **client** creates a transport connection to the **server** 
3. After connecting, the client discovers what tools the server offers (`get-alerts` and `get-forecast`)
4. When processing the query, the language model decides to use the `get-forecast` tool
5. The client executes the tool call, the server processes it, and returns weather data
6. Finally, the client disconnects from the server when the interaction is complete

This simple example demonstrates all three key components working together: the host managing user interaction and providing an interface, the client handling communication with the server, and the server providing specialized capabilities (weather data) that extend what the language model can do on its own.

Now that we've seen MCP in action, let's examine each component in detail.

### Core components: Hosts, clients, and servers

At the foundation of every MCP implementation are hosts, clients, and servers. Each plays a specific role in the overall architecture, creating a separation of concerns that enables modular design and flexible capability composition. This division of responsibilities creates natural boundaries for security, scaling, and development workflows.

MCP supports two distinct transport mechanisms that shape how these components interact:
1. **Stdio (process-based)** uses standard input/output streams for communication, with servers running as local processes spawned for each session
2. **SSE (HTTP-based)** uses server-sent events over HTTP for persistent connections, with servers running as long-lived services

These transport options influence how hosts manage servers and how clients establish connections. Let's explore how each component operates within this framework.

#### Hosts

Hosts serve as entry points for applications, functioning as the primary interface for users and orchestrating the overall interaction flow. Claude Desktop, Cursor, and the MCP CLI are examples of hosts, each with different user interfaces and feature sets. Hosts manage server configuration, create and coordinate clients, handle LLM selection, process user interactions, and transmit context between servers and language models.

The MCP CLI's `chat.ts` implementation demonstrates these host responsibilities:

```javascript
// From mcp-cli/src/commands/chat.ts - Host implementation
export function chatCommand(program: Command, config: any) {
  const chat = program
    .command('chat')
    .description('Start a chat session with an MCP server')
    .option('-s, --server <id>', 'Specify server ID to use')
    .option('-p, --provider <provider>', 'LLM provider to use (anthropic or openai)', 'anthropic')
    .option('-m, --model <model>', 'Model to use (defaults based on provider)')
    .action(async (options) => {
      // 1. Server configuration management
      const mcpServers = serverLib.getServerConfigs(config);
      const serverConfig = mcpServers[serverId];
      const isSSE = serverConfig.type === 'sse';
      
      // 2. LLM selection and configuration
      if (!options.model) {
        options.model = options.provider === 'anthropic' ? 
          'claude-3-5-sonnet-20241022' : 'gpt-4o';
      }
      
      // 3. Client creation and coordination
      client = new McpClient(serverId, `MCP Client (${serverId})`, {
        llmProvider: options.provider,
        model: options.model
      });
      
      // Connect to the server via client
      await client.connect(isSSE ? serverProcess : null, serverConfig);
      
      // 4. Processing user interactions
      const processQuery = async (query: string): Promise<void> => {
        try {
          logger.debug('Processing user query:', query);
          
          // 5. Transmitting context between servers and language models
          const response = await client.processQuery(query);
          
          console.log(chalk.green('Assistant: ') + response.trim());
        } catch (error) {
          logger.error('Error processing query:', error);
          console.log(chalk.red(`Error: ${error.message}`));
        }
      };
      
      // Set up input handling (interactive or piped)
      // ...
    });
}
```

An important responsibility of hosts is managing servers based on their type. For stdio servers, the host spawns and terminates processes as needed. For SSE servers, the host simply connects to existing server endpoints. This flexibility allows hosts to work with both local tools and remote services using the same protocol.

#### Clients

Clients maintain one-to-one connections with servers and handle the communication layer. Created and managed by hosts, clients translate between the host's needs and the server's capabilities. The `McpClient` class in the MCP CLI handles several key responsibilities:

The `connect` method orchestrates the connection process, creating an appropriate transport based on server type, establishing a connection, and discovering available tools. This is the entry point for client-server communication:

```javascript
// From mcp-cli/src/lib/mcp-client.ts
async connect(serverProcess: ChildProcess | null, serverConfig?: any): Promise<void> {
  // Create and initialize the client
  this.client = new Client({ name: this.name, version: '1.0.0' });
  
  // Create transport and establish connection
  this.transport = await this.createTransport(serverProcess, serverConfig);
  await this.client.connect(this.transport);
  
  // Discover available tools
  await this.discoverTools();
}
```

The `createTransport` method handles one of the client's core responsibilities: selecting the appropriate transport mechanism based on server type. This enables the protocol to work across different communication channels:

```javascript
// Creates either SSE or stdio transport based on server configuration
private async createTransport(serverProcess: ChildProcess | null, serverConfig?: any): Promise<Transport> {
  if (serverConfig?.type === 'sse') {
    // Create SSE transport for HTTP-based communication
    return new SSEClientTransport(new URL(serverConfig.url));
  } else {
    // Create stdio transport for process-based communication
    return new StdioClientTransport({
      command: serverConfig?.command,
      args: serverConfig?.args,
      process: serverProcess
    });
  }
}
```

After establishing the connection, the client discovers what tools the server offers through the `discoverTools` method. This dynamic discovery approach is fundamental to MCP's flexibility, allowing clients to adapt to whatever capabilities servers provide:

```javascript
// Requests and stores the list of available tools from the server
private async discoverTools(): Promise<void> {
  const toolsResponse = await this.client.listTools();
  this.tools = toolsResponse?.tools?.length ? toolsResponse.tools : [];
  this.logger.info(`Discovered ${this.tools.length} tools`);
}
```

The `callTool` method forms the core of client functionality, allowing the host to invoke server capabilities. It handles parameter validation, request formatting, and result processing:

```javascript
// Executes a tool with the provided arguments and processes the response
async callTool(toolName: string, args: Record<string, any>): Promise<string> {
  // Verify tool exists
  const tool = this.tools.find(t => t.name === toolName);
  if (!tool) throw new Error(`Tool "${toolName}" not found`);
  
  // Call the tool on the server
  const result = await this.client.callTool({ name: toolName, arguments: args });
  
  // Process and return the text content from the result
  return this.extractTextContent(result);
}
```

Finally, the `disconnect` method handles graceful shutdown, ensuring resources are properly released:

```javascript
// Gracefully terminates the connection and cleans up resources
async disconnect(): Promise<void> {
  if (this.client) await this.client.close();
  this.transport = null;
  this.client = null;
  this.isConnected = false;
  this.logger.info('Disconnected from server');
}
```

Through these methods, the client manages the complete lifecycle of server interaction, from connection establishment to discovery, execution, and termination. This structured approach ensures consistent communication patterns regardless of the underlying transport mechanism or server implementation.

#### Servers

Servers provide the actual tools, resources, and prompts that enhance language model capabilities. Servers come in two primary types, each suited to different use cases:

1. _Stdio servers_ use standard input/output streams for communication and run as local processes. These servers are spawned on demand by the host when a session begins and run as independent processes with their own lifecycle. They communicate with clients through stdin/stdout pipes, creating a direct channel for message exchange. When the session ends, stdio servers automatically terminate, ensuring resources are properly released. This approach provides perfect isolation between sessions since each client gets its own dedicated server process. The main limitation of stdio servers is that they're restricted to local execution, as they rely on process spawning within the same machine.

   Here's how hosts like the MCP CLI spawn stdio servers:

   ```javascript
   // Spawn a stdio server process
   if (!isSSE) {
   logger.debug(`Spawning stdio server using command: ${serverConfig.command}`);
   serverProcess = spawn(serverConfig.command, serverConfig.args || [], {
      env: { ...process.env, ...serverConfig.env },
      shell: true
   });

   // Forward server stderr output to client for debugging
   serverProcess.stderr.on('data', (data) => {
      const message = data.toString().trim();
      if (message) logger.debug(`Server stderr: ${message}`);
   });

   // Handle server process errors
   serverProcess.on('error', (error) => {
      logger.error(`Server process error: ${error.message}`);
   });
   }
   ```

1. _SSE servers_ use HTTP-based Server-Sent Events for persistent connections. Unlike stdio servers, they run as long-lived services independent of client sessions and can manage multiple client connections concurrently. They communicate over HTTP, using SSE for server-to-client messages and standard HTTP requests for client-to-server communication. SSE servers require explicit startup and shutdown rather than being automatically spawned for each session. They can maintain state between client connections, making them suitable for applications that need persistent context. Their network-based approach allows them to support remote access, enabling clients to connect to servers running on different machines or in cloud environments.

Stdio servers excel for simple, stateless tools like calculators or format converters, while SSE servers are better suited for stateful applications, multi-user systems, or remote services.

Part III will explore detailed server implementation strategies for both types.

### Server capabilities: Tools, resources, and prompts

MCP servers provide several distinct capabilities that extend language models in different ways. Each capability type serves a specific purpose in the architecture, with its own control pattern and integration approach.

- **Tools** enable language models to perform actions and modify state within controlled boundaries. Similar to POST endpoints in a REST API, tools remain model-controlled—designed for discovery and usage by the language model itself (with appropriate user approval). Each tool has a unique name, descriptive guidance, and a JSON Schema defining its parameters.

  A calculator server demonstrates this tool concept clearly:

  ```javascript
  // Example tool definition from calculator server
  server.tool(
    "add",                           // Tool name
    { a: z.number(), b: z.number() }, // Parameter schema using Zod
    async ({ a, b }) => {            // Implementation function
      const result = a + b;
      return {
        content: [{ type: "text", text: String(result) }]
      };
    }
  );
  ```

  More complex tools handle file operations, API integrations, or database queries—extending the model's capabilities beyond text generation.

- **Resources** function as data sources that LLMs can access and incorporate as context. Similar to GET endpoints in a REST API, these read-only providers remain application-controlled—the client application determines how and when to utilize them. Each resource has a unique URI (e.g., `weather://london/current`) and can contain text or binary data with appropriate MIME types.

  Resources extend beyond static files to include dynamic, real-time data. Servers can notify clients when resource content changes, creating a responsive context ecosystem.

- **Prompts** represent pre-defined templates that combine server data with structured formats to create consistent, contextually-appropriate content for the LLM. User-controlled rather than model-controlled, prompts are explicitly selected for specific purposes. Each prompt includes a name, description, optional arguments, and can incorporate embedded resources.

  The most powerful MCP implementations combine these three offerings effectively. A file editor server might implement file content access as resources, file modification as tools, and code review as prompts—creating a system where each type of functionality plays to its strengths.

### How clients and servers communicate

MCP follows a structured protocol that defines how components communicate and interact throughout a complete connection lifecycle. This standardized approach ensures consistent behavior across different implementations and transport mechanisms.

The complete lifecycle follows this sequence:

```
Client                                              Server
  |                                                   |
  |---------------initialize request----------------->|
  |                                                   |
  |<--------------initialize response----------------|
  |                                                   |
  |---------------initialized notification---------->|
  |                                                   |
  |                Connection ready                   |
  |                                                   |
  |----------------tools/list request--------------->|
  |                                                   |
  |<----------------tools list response--------------|
  |                                                   |
  |                  Tool discovery                   |
  |                                                   |
  |----------------tools/call request--------------->|
  |                                                   |
  |<-----------------tool execution result-----------|
  |                                                   |
  |                 Message exchange                  |
  |                                                   |
  |------------------shutdown request--------------->|
  |                                                   |
  |<------------------shutdown ack------------------|
  |                                                   |
  |                  Client disconnects               |
  |                                                   |
```

Let's examine each phase of this lifecycle in detail.

#### Initialization and discovery

Every MCP connection begins with a formal handshake sequence between client and server. This phase establishes compatibility and sets up the relationship before any actual work begins:

```javascript
// From the McpClient class - Connection initialization
async connect(serverProcess: ChildProcess | null, serverConfig?: any): Promise<void> {
  this.logger.info('Connecting to server...');
  
  try {
    // 1. Create the appropriate transport based on server type
    this.transport = await this.createTransport(serverProcess, serverConfig);
    
    // 2. Initialize the client with the transport
    this.client = new Client(this.transport);
    
    // 3. Send initialization request and establish connection
    await this.client.initialize({
      protocolVersion: '2024-11-05',
      capabilities: {},
      clientInfo: {
        name: this.name,
        version: '1.0.0',
        timeout: 60000
      }
    });
    
    // 4. Connection established
    this.isConnected = true;
    this.logger.info('Connected to server!');
    
    // 5. Request available tools from the server
    await this.requestTools();
  } catch (error) {
    this.logger.error('Failed to connect to server:', error);
    throw error;
  }
}
```

This initialization sequence serves several important purposes:
- It verifies protocol compatibility through version negotiation
- It establishes capability expectations between client and server
- It provides identification information for logging and troubleshooting

Only after this handshake succeeds does the client request the list of available tools, resources, and prompts through standardized discovery methods:

```javascript
// Client-side tool discovery
async requestTools() {
  this.logger.debug('Requesting tools list...');
  try {
    const response = await this.client.sendRequest('tools/list');
    this.tools = response.tools || [];
    this.logger.info(`Discovered ${this.tools.length} tools`);
    return this.tools;
  } catch (error) {
    this.logger.error('Failed to request tools:', error);
    throw error;
  }
}
```

#### Message exchange

Once the connection is established, MCP uses JSON-RPC 2.0 as its message format. Each request includes a method name, optional parameters, and a unique ID that correlates with its response:

```json
// Request format
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "add",
    "arguments": { "a": 5, "b": 7 }
  }
}

// Response format
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "12" }]
  }
}
```

The tool execution process showcases this pattern in action:

```javascript
// From the McpClient class - Tool execution
async callTool(toolName: string, args: any): Promise<any> {
  if (!this.isConnected) {
    throw new Error('Not connected to server');
  }
  
  this.logger.debug(`Executing tool call: ${toolName}`);
  
  try {
    // Send the tool call request via the client
    const result = await this.client.toolsCall(toolName, args);
    this.logger.debug(`Tool result:`, result);
    return result;
  } catch (error) {
    this.logger.error(`Tool call failed: ${error.message}`);
    throw error;
  }
}
```

#### Termination process (client disconnects)

The connection lifecycle concludes with a graceful termination sequence. This ensures all resources are properly released and both parties acknowledge the end of the session:

```javascript
// From the McpClient class - Disconnection handling
async disconnect(): Promise<void> {
  if (!this.isConnected) {
    return;
  }
  
  try {
    // Send a shutdown notification if we still have a connection
    if (this.client) {
      await this.client.shutdown();
    }
    
    // Clean up resources
    if (this.transport) {
      await this.transport.close();
      this.transport = null;
    }
    
    this.client = null;
    this.isConnected = false;
    this.tools = [];
    
    this.logger.info('Disconnected from server');
  } catch (error) {
    this.logger.error('Error during disconnect:', error);
    // Still mark as disconnected even if there was an error
    this.isConnected = false;
  }
}
```

This complete connection lifecycle—initialization, discovery, message exchange, and termination—creates a predictable interaction pattern that both clients and servers can rely on.

In Part III, we'll move from this conceptual understanding to practical implementation details for MCP servers. We'll examine the specific transport mechanisms (stdio vs SSE), security considerations, error handling strategies, and design patterns for building robust systems that effectively extend language model capabilities. 

## Part III: How to implement MCP servers?

Having explored the connection lifecycle and protocol flow in Part II, let's now examine how to implement these concepts in practical, production-ready code.

MCP supports two primary transport mechanisms: stdio for process-based communication and SSE for HTTP-based persistent connections. The choice between these options depends on specific application requirements:

When to use stdio transport:
- For stateless tools that don't need to maintain context between sessions
- When security isolation between clients is critical
- For simple use cases with minimal setup requirements
- When automatic lifecycle management is preferred

When to use SSE transport:
- For stateful applications that need to persist context across sessions
- When serving multiple clients concurrently is important
- For applications requiring remote access over networks
- When integration with existing HTTP infrastructure is needed

Now let's dive into how to implement each type of server.

### Create a stdio server

Stdio transport creates a communication channel between client and server using standard input/output streams. When implementing a stdio server, it's important to understand how hosts interact with these servers. For stdio servers, the host takes complete responsibility for the server lifecycle - it spawns a new server process for each client connection, manages the communication channels, and terminates the process when the session ends.

This lifecycle management means stdio servers don't need to implement their own connection handling or session management. Instead, they simply need to create a transport, connect it to the server, and let the host handle the rest. This simplifies the server implementation but means the server runs as a new process for each client.

Here's the real implementation from the calculator stdio server. This calculator server implements four basic mathematical operations as tools. Each tool follows the same pattern: it accepts two numbers, performs an operation, and returns the result as text. Note that the division tool includes error handling for division by zero. The server initialization process is straightforward - it creates a stdio transport and connects it to the server.

```javascript
// From mcp-calculator-server/index.js
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// Create a simple MCP server
const server = new McpServer({
  name: "Calculator Server",
  version: "1.0.0"
});

// Add addition tool
server.tool(
  "add",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => {
    const result = a + b;
    return {
      content: [{ type: "text", text: String(result) }]
    };
  }
);

// Truncated...

// Add division tool
server.tool(
  "divide",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => {
    if (b === 0) {
      throw new Error("Cannot divide by zero");
    }
    const result = a / b;
    return {
      content: [{ type: "text", text: String(result) }]
    };
  }
);

// Start the server
async function main() {
  // Create a transport
  const transport = new StdioServerTransport();
  
  // Connect the server to the transport
  await server.connect(transport);
  
  console.error("Calculator MCP server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

The stdio approach offers significant advantages for certain use cases. The process isolation creates natural security boundaries, with each client receiving a dedicated server instance. The automatic lifecycle management simplifies server administration—processes start when needed and terminate when the connection ends. Stdio transport also has minimal dependencies, making it easier to implement and deploy in various environments.

However, stdio transport has important limitations. The process-per-client model means each client gets its own server process, which prevents state persistence between different client sessions, though a single client can maintain context throughout its entire conversation. 

One significant practical challenge with stdio servers is debugging. Since the stdout stream is used for protocol communication, you can't use standard console.log statements as they would interfere with the message exchange. Instead, developers must use alternative logging approaches such as writing to stderr (which doesn't interfere with the protocol) or logging to files:

```javascript
// Example of stdio-safe logging
function log(message) {
  // Log to stderr instead of stdout
  console.error(`[DEBUG] ${message}`);
  
  // Or write to a log file
  fs.appendFileSync('server.log', `${new Date().toISOString()} - ${message}\n`);
}

// Usage in a tool
server.tool("example", { input: z.string() }, async ({ input }) => {
  log(`Processing input: ${input}`);
  // Process normally...
});
```

This debugging limitation might seem minor, but it can significantly impact development workflow and error diagnosis in production environments.

#### Prevent security vulnerabilities

Security is critical in MCP servers, especially when handling file access or executing commands:

```javascript
// Path traversal prevention example
function validateFilePath(requestedPath: string, allowedRoot: string): string {
  // Normalize to absolute path
  const absolutePath = path.isAbsolute(requestedPath)
    ? requestedPath
    : path.join(process.cwd(), requestedPath);
  
  // Normalize to remove .. segments
  const normalizedPath = path.normalize(absolutePath);
  
  // Check if within allowed directory
  if (!normalizedPath.startsWith(allowedRoot)) {
    throw new Error(`Access denied: Path is outside allowed directory`);
  }
  
  return normalizedPath;
}

// Example of secure file reading tool
server.tool(
  "readFile",
  { path: z.string() },
  async ({ path: filePath }) => {
    try {
      // Validate path before using
      const validatedPath = validateFilePath(filePath, process.cwd());
      
      // Read file only after validation
      const content = await fs.promises.readFile(validatedPath, 'utf8');
      return { content: [{ type: "text", text: content }] };
    } catch (error) {
      return {
        content: [{ type: "text", text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);
```

This implementation prevents path traversal attacks by validating that requested paths stay within allowed directories. Similar precautions should be taken for any operation that accesses external resources.

### Create an SSE Server

Unlike stdio servers where hosts manage the complete lifecycle, SSE servers operate independently of the host. When implementing an SSE server, the server itself is responsible for maintaining its own lifecycle, handling multiple client connections concurrently, and managing session state. The host simply connects to the server's HTTP endpoint and communicates via the SSE protocol.

This means SSE servers must implement their own connection management, session tracking, and communication handling. They typically run continuously as standalone services, waiting for clients to connect. Hosts don't spawn or terminate SSE servers - instead, they discover and connect to existing server endpoints.

An SSE server uses HTTP to handle connections and message exchange. Here's the real implementation from the calculator SSE server.

The calculator SSE server implements the same four mathematical operations as tools, but with a significantly different transport implementation. It uses Express.js to create an HTTP server with two main endpoints:

1. A GET endpoint at `/sse` that establishes the SSE connection and creates a transport for each client
2. A POST endpoint at `/messages` that receives JSON-RPC messages from clients

The server maintains a map of active transports indexed by session ID, creating a new transport for each client connection and removing it when the connection closes. This session management is a key difference from stdio servers, allowing the SSE server to handle multiple concurrent clients.

```javascript
// From mcp-calculator-sse-server/index.js
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import express from 'express';
import cors from 'cors';
import { z } from "zod";

const app = express();
app.use(cors());
app.use(express.json());

// Add request logging
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} ${req.method} ${req.url}`);
  next();
});

const PORT = process.env.PORT || 3000;
const NAME = "Calculator Server";
const VERSION = "1.0.0";

// Create a simple MCP server
const server = new McpServer({
  name: NAME,
  version: VERSION
});

// Add addition tool
server.tool(
  "add",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => {
    const result = a + b;
    return {
      content: [{ type: "text", text: String(result) }]
    };
  }
);

// Truncated...

// Add division tool
server.tool(
  "divide",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => {
    if (b === 0) {
      throw new Error("Cannot divide by zero");
    }
    const result = a / b;
    return {
      content: [{ type: "text", text: String(result) }]
    };
  }
);

// Store active transports by session ID
const activeTransports = new Map();

// SSE endpoint handler
app.get('/sse', async (req, res) => {
  console.log('SSE connection attempt received');
  
  // Create SSE transport for this connection
  const transport = new SSEServerTransport('/messages', res);
  const sessionId = req.query.sessionId || transport.sessionId;
  console.log(`Created transport with session ID: ${sessionId}`);
  
  // Store the transport
  activeTransports.set(sessionId, transport);
  
  // Clean up on connection close
  req.on('close', () => {
    console.log(`Cleaning up transport for session ${sessionId}`);
    activeTransports.delete(sessionId);
  });
  
  try {
    // Connect transport to server
    await server.connect(transport);
    console.log('Server connected to SSE transport');
  } catch (error) {
    console.error('Error connecting transport:', error);
    if (!res.headersSent) {
      res.status(500).end();
    }
  }
});

// Handle messages
app.post('/messages', express.json(), async (req, res) => {
  const sessionId = req.query.sessionId;
  console.log(`Received SSE client message for session ${sessionId}:`, req.body);
  
  try {
    // Get the transport for this session
    const transport = activeTransports.get(sessionId);
    if (!transport) {
      throw new Error(`No active transport found for session ${sessionId}`);
    }

    // Let the transport handle the message routing
    const response = await transport.handleMessage(req.body);
    res.json(response);
  } catch (error) {
    console.error('Error handling SSE client message:', error);
    res.status(500).json({ error: error.message });
  }
});

// Start the server
async function main() {
  app.listen(PORT, () => {
    console.log(`Calculator MCP server running on http://localhost:${PORT}`);
    console.log("Server is running. Press Ctrl+C to stop.");
  });

  // Handle process termination
  process.on('SIGINT', () => {
    console.log('Received SIGINT. Shutting down...');
    process.exit(0);
  });

  process.on('SIGTERM', () => {
    console.log('Received SIGTERM. Shutting down...');
    process.exit(0);
  });
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

The SSE approach offers several compelling advantages for certain applications. The HTTP-based communication enables remote accessibility, allowing clients to connect to servers running on different machines or in cloud environments. SSE servers can also handle multiple concurrent connections through a single server process, which can be more resource-efficient than spawning separate processes for each client.

However, these benefits come with significant additional complexity, especially in production environments. The example shown stores session transports in an in-memory Map:

```javascript
// Store active transports by session ID
const activeTransports = new Map();

// Store the transport
activeTransports.set(sessionId, transport);
```

This approach, while simple for demonstration purposes, is not production-ready. In a real-world deployment, in-memory storage fails when scaling to multiple server instances behind a load balancer, doesn't persist across server restarts, and risks memory leaks if connections aren't properly cleaned up. For production use, you would need an external session storage solution like Redis to maintain transport and session information across multiple server instances, ensuring clients can reconnect even if they're load-balanced to a different server or if the original server restarts.

The HTTP dependencies also introduce additional requirements for deployment and configuration. The persistent nature of SSE servers means they must be started and stopped explicitly rather than spawning automatically when needed, adding operational complexity.

### SSE server lifecycle

Unlike stdio servers that are automatically managed by the host, SSE servers require explicit lifecycle management. Here's how to manage an SSE server using the MCP CLI:

1. Start the server.

    ```bash
    ❯ mcp server start calculator-sse -d
    2025-03-17T06:42:23.153Z [DEBUG] server: Server calculator-sse started successfully (PID: 43687)
    ✔ Server calculator-sse started successfully (PID: 43687).
    Server is running in the background.
    Server logs are being written to /Users/dos/mcp/logs/server-calculator-sse.log
    To stop the server, run: mcp server stop calculator-sse
    To chat with the server, run: mcp chat --server calculator-sse
    ```

2. Send a query to the server.

    ```bash
    ❯ echo "what is 20+22?" | mcp chat -s calculator-sse -d
    2025-03-17T06:42:34.078Z [INFO]  client:calculator-sse: Creating MCP client: calculator-sse (MCP Client (calculator-sse))
    2025-03-17T06:42:34.079Z [INFO]  client:calculator-sse: Creating MCP client...
    2025-03-17T06:42:34.079Z [DEBUG] client:calculator-sse: Server process:
    2025-03-17T06:42:34.079Z [DEBUG] client:calculator-sse: Creating transport...
    2025-03-17T06:42:34.079Z [DEBUG] client:calculator-sse: Using SSE transport with URL: http://localhost:3000
    2025-03-17T06:42:35.081Z [DEBUG] client:calculator-sse: Transport created successfully
    2025-03-17T06:42:35.083Z [INFO]  client:calculator-sse: Connecting to server...
    2025-03-17T06:42:35.145Z [INFO]  client:calculator-sse: Connected to server!
    2025-03-17T06:42:35.145Z [DEBUG] client:calculator-sse: Requesting tools list...
    2025-03-17T06:42:35.149Z [DEBUG] client:calculator-sse: Raw tools response: {"tools":[{"name":"add","inputSchema":{"type":"object","properties":{"a":{"type":"number"},"b":{"type":"number"}},"required":["a","b"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}},{"name":"subtract","inputSchema":{"type":"object","properties":{"a":{"type":"number"},"b":{"type":"number"}},"required":["a","b"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}},{"name":"multiply","inputSchema":{"type":"object","properties":{"a":{"type":"number"},"b":{"type":"number"}},"required":["a","b"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}},{"name":"divide","inputSchema":{"type":"object","properties":{"a":{"type":"number"},"b":{"type":"number"}},"required":["a","b"],"additionalProperties":false,"$schema":"http://json-schema.org/draft-07/schema#"}}]}
    2025-03-17T06:42:35.149Z [INFO]  client:calculator-sse: Discovered 4 tools
    2025-03-17T06:42:35.150Z [INFO]  client:calculator-sse: Available tools: add, subtract, multiply, divide
    2025-03-17T06:42:35.150Z [DEBUG] host:chat: Starting chat session with server: calculator-sse
    2025-03-17T06:42:35.150Z [DEBUG] host:chat: Starting non-interactive chat session (piped input)
    2025-03-17T06:42:35.151Z [DEBUG] host:chat: Processing piped query:
    2025-03-17T06:42:35.151Z [DEBUG] client:calculator-sse: Processing query: what is 20+22?
    2025-03-17T06:42:35.152Z [DEBUG] client:calculator-sse: Available tools:
    2025-03-17T06:42:38.602Z [DEBUG] client:calculator-sse: Executing tool call: add
    2025-03-17T06:42:38.602Z [DEBUG] client:calculator-sse: Calling tool "add" with args:
    2025-03-17T06:42:38.609Z [DEBUG] client:calculator-sse: Tool result: 42
    20 + 22 = 42
    2025-03-17T06:42:39.412Z [DEBUG] host:chat: Disconnecting client during shutdown
    2025-03-17T06:42:39.412Z [INFO]  client:calculator-sse: Disconnecting from server
    2025-03-17T06:42:39.414Z [INFO]  client:calculator-sse: Disconnected from server
    2025-03-17T06:42:39.414Z [DEBUG] host:chat: Chat session ended
    ```

3. Check server logs.
    
    ```bash
    ❯ cat ~/mcp/logs/server-calculator-sse.log
    [2025-03-17T06:42:23.138Z] === SERVER START ===
    Command: node /Users/dos/Desktop/mcp/mcp-servers/mcp-calculator-sse-server/index.js
    Working directory: /Users/dos/Desktop/mcp/mcp-servers/mcp-calculator-sse-server
    Type: sse
    URL: http://localhost:3000

    Calculator MCP server running on http://localhost:3000
    Server is running. Press Ctrl+C to stop.
    Available endpoints:
      - GET / (SSE endpoint)
      - POST * (message endpoint)
    Registered route: get /*
    Registered route: post *
    2025-03-17T06:42:35.119Z GET /
    SSE connection attempt received
    Created transport with session ID: 55984849-ee5e-4f65-bace-fc45d0d398fa
    Server connected to SSE transport
    2025-03-17T06:42:35.138Z POST /messages?sessionId=55984849-ee5e-4f65-bace-fc45d0d398fa
    Received SSE client message for session 55984849-ee5e-4f65-bace-fc45d0d398fa: {
      method: 'initialize',
      params: {
        protocolVersion: '2024-11-05',
        capabilities: {},
        clientInfo: {
          name: 'MCP Client (calculator-sse)',
          version: '1.0.0',
          timeout: 60000
        }
      },
      jsonrpc: '2.0',
      id: 0
    }
    2025-03-17T06:42:35.144Z POST /messages?sessionId=55984849-ee5e-4f65-bace-fc45d0d398fa
    Received SSE client message for session 55984849-ee5e-4f65-bace-fc45d0d398fa: { method: 'notifications/initialized', jsonrpc: '2.0' }
    2025-03-17T06:42:35.147Z POST /messages?sessionId=55984849-ee5e-4f65-bace-fc45d0d398fa
    Received SSE client message for session 55984849-ee5e-4f65-bace-fc45d0d398fa: { method: 'tools/list', jsonrpc: '2.0', id: 1 }
    2025-03-17T06:42:38.607Z POST /messages?sessionId=55984849-ee5e-4f65-bace-fc45d0d398fa
    Received SSE client message for session 55984849-ee5e-4f65-bace-fc45d0d398fa: {
      method: 'tools/call',
      params: { name: 'add', arguments: { a: 20, b: 22 } },
      jsonrpc: '2.0',
      id: 2
    }
    Cleaning up transport for session 55984849-ee5e-4f65-bace-fc45d0d398fa
   ```

4. Stop the server.

    ```bash
    ❯ mcp server stop calculator-sse -d
    ⠦ Stopping server...2025-03-17T06:46:06.873Z [INFO]  server: Process 43871 was successfully terminated
    2025-03-17T06:46:06.891Z [DEBUG] server: Server "calculator-sse" stopped successfully
    ✔ Server "calculator-sse" stopped successfully.
    ```

This explicit lifecycle management is a key difference from stdio servers, which are automatically created and destroyed for each client connection. With SSE servers, you're responsible for starting the server before clients can connect to it and stopping it when it's no longer needed.

## Conclusion

The field of MCP implementation continues to evolve, with ongoing developments in:

- Authentication and access control
- Real-time streaming capabilities
- Cross-server federation
- Standardized testing frameworks

By building on the patterns and practices described here, you can create robust, secure, and scalable MCP servers that extend language models with specialized capabilities while maintaining the flexibility to adapt as the protocol evolves. 