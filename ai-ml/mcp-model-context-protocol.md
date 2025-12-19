# Model Context Protocol (MCP)

Anthropic's open protocol for connecting AI assistants to external data sources and tools.

## What is MCP?

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP ARCHITECTURE                              │
│                                                                 │
│   ┌──────────────┐        ┌──────────────┐                     │
│   │   AI Host    │        │  MCP Server  │                     │
│   │  (Claude)    │◄──────►│  (Your App)  │                     │
│   └──────────────┘  JSON  └──────────────┘                     │
│         │           RPC          │                              │
│         │                        │                              │
│         ▼                        ▼                              │
│   ┌──────────────┐        ┌──────────────┐                     │
│   │   Context    │        │  Resources   │                     │
│   │   Window     │        │  - Files     │                     │
│   │              │        │  - Database  │                     │
│   └──────────────┘        │  - APIs      │                     │
│                           └──────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## MCP Server Implementation

```typescript
// server.ts - Basic MCP Server
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  {
    name: "my-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      resources: {},
      tools: {},
    },
  }
);

// Define available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "search_database",
        description: "Search the product database",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "Search query",
            },
            limit: {
              type: "number",
              description: "Max results to return",
              default: 10,
            },
          },
          required: ["query"],
        },
      },
      {
        name: "create_ticket",
        description: "Create a support ticket",
        inputSchema: {
          type: "object",
          properties: {
            title: { type: "string" },
            description: { type: "string" },
            priority: {
              type: "string",
              enum: ["low", "medium", "high"],
            },
          },
          required: ["title", "description"],
        },
      },
    ],
  };
});

// Handle tool execution
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "search_database":
      const results = await searchDatabase(args.query, args.limit);
      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(results, null, 2),
          },
        ],
      };

    case "create_ticket":
      const ticket = await createTicket(args);
      return {
        content: [
          {
            type: "text",
            text: `Created ticket #${ticket.id}: ${ticket.title}`,
          },
        ],
      };

    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});

// Define available resources
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return {
    resources: [
      {
        uri: "file:///config/settings.json",
        name: "Application Settings",
        mimeType: "application/json",
      },
      {
        uri: "db://users/schema",
        name: "Users Database Schema",
        mimeType: "application/json",
      },
    ],
  };
});

// Handle resource reads
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  if (uri === "file:///config/settings.json") {
    const settings = await readSettings();
    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: JSON.stringify(settings, null, 2),
        },
      ],
    };
  }

  if (uri === "db://users/schema") {
    const schema = await getDatabaseSchema("users");
    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: JSON.stringify(schema, null, 2),
        },
      ],
    };
  }

  throw new Error(`Resource not found: ${uri}`);
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch(console.error);
```

## Claude Desktop Configuration

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/path/to/server.js"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb",
        "API_KEY": "your-api-key"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/me/projects"
      ]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_xxxx"
      }
    }
  }
}
```

## Common MCP Servers

### Filesystem Server

```bash
# Install and run
npx -y @modelcontextprotocol/server-filesystem /path/to/allowed/directory
```

Capabilities:
- Read files
- Write files
- List directories
- Search files

### PostgreSQL Server

```bash
npx -y @modelcontextprotocol/server-postgres postgresql://localhost/mydb
```

Capabilities:
- Query database
- List tables
- Get schema

### GitHub Server

```bash
GITHUB_TOKEN=ghp_xxx npx -y @modelcontextprotocol/server-github
```

Capabilities:
- Search repositories
- Read files
- Create issues
- Create pull requests

## Building Custom MCP Tools

```typescript
// Translation MCP Server Example
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const translationServer = new Server({
  name: "translation-server",
  version: "1.0.0",
});

// Add translation key tool
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "translate_add_key") {
    const { key, english_text, translations } = request.params.arguments;

    // Save to translation files
    for (const [locale, text] of Object.entries(translations)) {
      await saveTranslation(locale, key, text);
    }

    return {
      content: [
        {
          type: "text",
          text: `Added translation key "${key}" to ${Object.keys(translations).length} locales`,
        },
      ],
    };
  }
});

// List translation keys
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const keys = await getAllTranslationKeys();
  return {
    resources: keys.map((key) => ({
      uri: `translation://${key}`,
      name: key,
      mimeType: "application/json",
    })),
  };
});
```

## MCP Client Usage

```typescript
// Using MCP in your application
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const client = new Client({
  name: "my-client",
  version: "1.0.0",
});

// Connect to MCP server
const transport = new StdioClientTransport({
  command: "node",
  args: ["server.js"],
});

await client.connect(transport);

// List available tools
const tools = await client.listTools();
console.log("Available tools:", tools);

// Call a tool
const result = await client.callTool({
  name: "search_database",
  arguments: { query: "typescript", limit: 5 },
});

console.log("Search results:", result);

// Read a resource
const resource = await client.readResource({
  uri: "file:///config/settings.json",
});

console.log("Settings:", resource);
```

## Best Practices

1. **Security**: Validate all inputs, use environment variables for secrets
2. **Error Handling**: Return clear error messages, don't expose internal details
3. **Performance**: Cache frequently accessed resources, implement pagination
4. **Documentation**: Provide clear descriptions for tools and resources
5. **Testing**: Test tool handlers independently before integration

---

*Learned: December 20, 2025*
*Tags: MCP, Model Context Protocol, Claude, AI, Tools, Anthropic*
