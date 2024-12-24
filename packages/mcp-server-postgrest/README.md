# @supabase/mcp-server-postgrest

This is an MCP server for PostgREST. It allows LLMs perform database queries and operations on Postgres databases via PostgREST.

## Usage

### With Claude Desktop

[Claude Desktop](https://claude.ai/download) is a popular LLM client that supports the Model Context Protocol. You can connect your PostgREST server to Claude Desktop to query your database via natural language commands.

You can add MCP servers to Claude Desktop via its config file at:

- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`

- Windows:`%APPDATA%\Claude\claude_desktop_config.json`

To add your Supabase project _(or any PostgREST server)_ to Claude Desktop, add the following configuration to the `servers` array in the config file:

```json
{
  "mcpServers": {
    "todos": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-postgrest",
        "--apiUrl",
        "https://your-project-ref.supabase.co/rest/v1",
        // or http://127.0.0.1:54321/rest/v1 for local
        "--apiKey",
        "your-anon-key",
        // or service role key, or user JWT
        "--schema",
        "public"
        // or any other exposed schema
      ]
    }
  }
}
```

### Programmatically (custom MCP client)

If you're building your own MCP client, you can connect to a PostgREST server programmatically using your preferred transport. The [MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk) offers built-in [stdio](https://modelcontextprotocol.io/docs/concepts/transports#standard-input-output-stdio) and [SSE](https://modelcontextprotocol.io/docs/concepts/transports#server-sent-events-sse) transports. We also offer a [`StreamTransport`](../mcp-utils/) if you wish to directly connect to MCP servers in-memory or over your own stream-based transport.

#### Installation

```bash
npm i @supabase/mcp-server-postgrest
```

```bash
yarn add @supabase/mcp-server-postgrest
```

```bash
pnpm add @supabase/mcp-server-postgrest
```

#### Example

The following example uses the [`StreamTransport`](../mcp-utils/) to connect directly between an MCP client and server.

```ts
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamTransport } from '@supabase/mcp-utils';
import { PostgrestMcpServer } from '@supabase/mcp-server-postgrest';

// Create a stream transport for both client and server
const clientTransport = new StreamTransport();
const serverTransport = new StreamTransport();

// Connect the streams together
clientTransport.readable.pipeTo(serverTransport.writable);
serverTransport.readable.pipeTo(clientTransport.writable);

const client = new Client(
  {
    name: 'MyClient',
    version: '0.1.0',
  },
  {
    capabilities: {},
  }
);

const supabaseUrl = 'https://your-project-ref.supabase.co'; // http://127.0.0.1:54321 for local
const apiKey = 'your-anon-key'; // or service role, or user JWT
const schema = 'public'; // or any other exposed schema

const server = new PostgrestMcpServer({
  apiUrl: `${supabaseUrl}/rest/v1`,
  apiKey,
  schema,
});

// Connect the client and server to their respective transports
await server.connect(serverTransport);
await client.connect(clientTransport);

// Call tools, etc
const output = await client.callTool({
  name: 'postgrestRequest',
  arguments: {
    method: 'GET',
    path: '/todos',
  },
});
```

### Configuration

- `apiUrl`: The base URL of your PostgREST endpoint

- `apiKey`: Your API key for authentication _(optional)_

- `schema`: The Postgres schema to serve the API from. Note any non-public schemas must be manually exposed from PostgREST.