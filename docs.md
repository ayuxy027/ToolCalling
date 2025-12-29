# Function Calling with Ollama FunctionGemma

## Prerequisites

Install Ollama and pull the functiongemma model:
```bash
ollama pull functiongemma
```

Ensure Ollama is running on http://localhost:11434 (or set OLLAMA_HOST environment variable).

## Setup

Define your tool functions. Each function MUST return a JSON string:

```typescript
function yourFunction(param1: string, param2: number): string {
  return JSON.stringify({ result: yourComputation });
}
```

## Tool Definition Structure

Define tools as an array of function descriptors. Each tool MUST follow this structure:

```typescript
const tools = [
  {
    type: 'function',
    function: {
      name: 'function_name',           // Must match your function name
      description: 'Clear description of what the function does.',
      parameters: {
        type: 'object',
        properties: {
          paramName: { 
            type: 'string' | 'number', 
            description: 'Parameter description' 
          },
        },
        required: ['paramName'],      // List all required parameters
      },
    },
  },
];
```

## Message Interface

Define interfaces for type safety:

```typescript
interface Message {
  role: 'user' | 'assistant' | 'tool';
  content: string;
  tool_calls?: { 
    id?: string;
    function: { 
      name: string; 
      arguments: Record<string, any> | string;
    } 
  }[];
  tool_call_id?: string;
}

interface ChatResponse {
  message: Message;
}
```

## Chat API Call

Send messages to Ollama with tools enabled:

```typescript
async function chat(messages: Message[]): Promise<ChatResponse> {
  const response = await fetch(`${OLLAMA_HOST}/api/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ 
      model: 'functiongemma',
      messages,
      tools,
      stream: false 
    }),
  });

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  return response.json();
}
```

## Tool Execution Handler

Create a switch statement to route tool calls to your functions:

```typescript
function executeTool(name: string, args: Record<string, any>): string {
  switch (name) {
    case 'function_name':
      return yourFunction(args.param1, Number(args.param2));
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
}
```

## Tool Call Processing

Process tool calls from the model response:

```typescript
const response = await chat(messages);

if (response.message.tool_calls?.length) {
  // Add assistant message with tool calls
  messages.push(response.message);
  
  // Execute each tool call
  for (const tool of response.message.tool_calls) {
    // Parse arguments (may be string or object)
    const args = typeof tool.function.arguments === 'string' 
      ? JSON.parse(tool.function.arguments) 
      : tool.function.arguments;
    
    // Execute the tool
    const result = executeTool(tool.function.name, args);
    
    // Add tool response to messages
    messages.push({ 
      role: 'tool', 
      content: result, 
      tool_call_id: tool.id || tool.function.name 
    });
  }
  
  // Get final response from model
  const final = await chat(messages);
}
```

## FunctionGemma Best Practices

1. **Tool Descriptions**: Write clear, specific descriptions. The model uses these to select tools.

2. **Parameter Types**: Always specify correct types ('string' or 'number'). FunctionGemma respects these types.

3. **Required Parameters**: List all required parameters in the `required` array. Omit optional parameters.

4. **Function Return Format**: All functions MUST return JSON strings. Use `JSON.stringify()`.

5. **Argument Parsing**: Always handle both string and object argument formats from the model.

6. **Type Conversion**: Convert string arguments to numbers when needed using `Number()`.

7. **Tool Naming**: Use snake_case for tool names. Match function names exactly in executeTool switch.

8. **Multiple Tool Calls**: Handle multiple tool calls in a single response. Process all before sending final chat request.

9. **Error Handling**: Validate tool arguments before execution. Throw descriptive errors for unknown tools.

10. **Message Flow**: Maintain conversation history: user message → assistant (with tool_calls) → tool responses → final assistant response.

## Running

Execute with Bun:
```bash
bun run tool.ts "Your prompt here"
```

Or with Node/TSX:
```bash
npx tsx tool.ts "Your prompt here"
```
