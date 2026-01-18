# LLM Integration & Streaming

## Component Role

The LLM integration layer connects Loom to language model providers like Anthropic's Claude and OpenAI's GPT. It provides a unified interface so the rest of Loom doesn't need to know about provider-specific APIs. This layer also handles streaming responses so you see text appear in real-time instead of waiting for the entire response.

## Referenced Components

**Agent State Machine**: Uses LlmClient to make completion requests and processes streaming LlmEvents.
**Thread System**: Threads record which provider and model were used.
**Tool System**: Tool definitions are sent to the LLM so it knows what functions it can call.
**Server**: Acts as a proxy between clients and LLM providers, keeping API keys server-side.

## How It Works

### LlmClient Trait

At the foundation is the LlmClient trait, which defines two operations:

**complete** - Send a request and wait for the full response. This is simpler but provides no feedback until the LLM finishes generating.

**complete_streaming** - Send a request and get back a stream of events. Text and tool calls arrive incrementally as they're generated.

Both methods are async and take an LlmRequest containing the model name, conversation messages, available tools, and optional parameters like max_tokens and temperature.

The trait uses async_trait because Rust doesn't yet support async functions in traits natively. This macro transforms the async fn into a regular function returning a pinned boxed future, enabling trait objects.

### Provider Architecture: Server-Side Proxy

Loom uses a server-side proxy architecture where API keys live only on the server. Clients never see or handle provider credentials. Here's how it flows:

1. The CLI or web UI creates a ProxyLlmClient pointing at the server URL.
2. The client chooses a provider (anthropic or openai) when constructing the client.
3. Requests go to provider-specific endpoints: /proxy/anthropic/complete or /proxy/openai/stream.
4. The server's LlmService receives the request and uses its stored API key to make the actual provider API call.
5. Responses stream back through the server to the client.

This design has key benefits:
- Clients don't need API keys—just authentication to the server
- API keys can be rotated server-side without updating clients
- Server can log and audit all LLM usage
- Easier to switch providers (just update server config)

### ProxyLlmClient

The ProxyLlmClient lives in the loom-llm-proxy crate and implements LlmClient by making HTTP requests to the server. It's provider-aware—you construct it with an explicit provider choice:

```
ProxyLlmClient::anthropic(server_url)
ProxyLlmClient::openai(server_url)
```

For the complete method, it POSTs the LlmRequest JSON to /proxy/{provider}/complete and deserializes the JSON response into LlmResponse.

For complete_streaming, it POSTs to /proxy/{provider}/stream and parses the Server-Sent Events response into a stream of LlmEvents. The SSE parsing handles the data: prefix format and deserializes each event into the unified LlmStreamEvent wire format.

### LlmService

On the server side, the LlmService wraps the actual provider clients (AnthropicClient and OpenAIClient). It can have both providers configured simultaneously by reading API keys from environment variables:
- ANTHROPIC_API_KEY for Claude
- OPENAI_API_KEY for GPT

The service exposes provider-specific methods:
- has_anthropic() / has_openai() - Check if a provider is configured
- complete_anthropic() / complete_openai() - Non-streaming completion
- complete_streaming_anthropic() / complete_streaming_openai() - Streaming completion

The server's HTTP handlers check which provider was requested via the URL path, then call the appropriate LlmService method.

### Provider Clients (Server-Only)

The actual provider implementations live in loom-llm-anthropic and loom-llm-openai. These crates are server-only dependencies—the CLI never imports them. They implement the LlmClient trait using provider-specific HTTP APIs.

**AnthropicClient** talks to api.anthropic.com using the Messages API. It sends POST requests to /v1/messages with:
- x-api-key header with the API key
- anthropic-version header (currently "2023-06-01")
- JSON body with model, messages array, tool definitions
- System messages extracted to a top-level system field

**OpenAIClient** talks to api.openai.com/v1 using the Chat Completions API. It sends POST requests to /chat/completions with:
- Authorization: Bearer {api_key} header
- JSON body with model, messages array, tools array
- Optional OpenAI-Organization header for team accounts

Both clients handle provider-specific quirks:
- Anthropic uses tool_result content blocks for tool results
- OpenAI uses separate tool role messages with tool_call_id
- Token usage field names differ (input_tokens vs prompt_tokens)

### Message Format Conversion

The loom-core Message struct is provider-agnostic:
- role: System, User, Assistant, or Tool
- content: The text content
- tool_call_id: For Tool role, which tool call this result belongs to
- name: For Tool role, the tool name

Each provider client converts these to its native format:

**Anthropic** pulls out System messages to a separate system field. Tool result messages become user messages with tool_result content blocks containing the tool_use_id and output.

**OpenAI** keeps System as a message role. Tool result messages use the tool role with tool_call_id and name fields.

These conversions are encapsulated in the provider client, so the rest of Loom works with one consistent format.

### Streaming: Server-Sent Events

Streaming uses the Server-Sent Events (SSE) standard, which is simpler than WebSockets for one-directional server-to-client data flow. SSE messages have the format:

```
event: message_type
data: {"json": "payload"}

event: another_type
data: {"more": "data"}
```

Each event is separated by double newlines. Lines starting with "event:" specify the event type. Lines starting with "data:" contain the payload. Lines starting with ":" are comments (often used for keep-alive pings).

### LlmEvent Union

The LlmEvent enum unifies streaming events across all providers:

**TextDelta** - A fragment of the assistant's text response. Contains just the content string. Multiple TextDelta events are accumulated to build the complete message.

**ToolCallDelta** - Partial information about a tool call. Contains call_id (unique identifier), tool_name (which tool to invoke), and arguments_fragment (a piece of the JSON arguments).

**Completed** - The stream has finished successfully. Contains the complete LlmResponse with the final message, all tool calls with parsed arguments, usage statistics, and finish reason.

**Error** - Something went wrong during streaming. Contains the LlmError with details about what failed.

The agent processes these events in its CallingLlm state, forwarding TextDelta events for immediate display and accumulating state until Completed or Error arrives.

### Tool Call Accumulation

Tool arguments stream as JSON fragments that may be syntactically incomplete:

First event: `{"loc`
Second event: `ation": "`
Third event: `NYC"}`

You can't parse these fragments individually—they're not valid JSON. Instead, they must be accumulated in a string buffer until the tool call completes. Only then is the full JSON string parsed into a serde_json::Value.

Each provider's stream parser maintains a HashMap from tool call index or ID to a ToolCallBuilder that accumulates the name and arguments. When the stream indicates the tool call is complete (content_block_stop for Anthropic, or the finish event for OpenAI), the builder parses the accumulated arguments and creates a ToolCall.

### Anthropic SSE Format

Anthropic's streaming API emits a sequence of events that form a state machine:

1. message_start - Stream begins with message ID, model, and input token count
2. content_block_start - A new content block (text or tool_use) begins with its type and metadata
3. content_block_delta - Incremental content arrives (text_delta or input_json_delta)
4. content_block_stop - The content block is complete
5. message_delta - Message-level updates like stop_reason and output token count
6. message_stop - Stream is complete
7. ping - Keep-alive signal (just a comment, no action needed)
8. error - API error occurred

Text and tool calls can be interleaved. Multiple content blocks can stream in sequence. The parser tracks the current block index and maintains separate builders for each tool call.

### OpenAI SSE Format

OpenAI uses a simpler line-based format where every payload line starts with "data:":

```
data: {"id":"chatcmpl-123","choices":[{"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl-123","choices":[{"delta":{"content":" world"}}]}

data: [DONE]
```

The literal string "data: [DONE]" signals stream completion. This isn't valid JSON, so it requires special handling—the parser checks for this exact string before trying to parse JSON.

Tool calls arrive as deltas with an index field indicating which tool call. Multiple tool calls can stream concurrently, each with its own index. The parser maintains a HashMap from index to AccumulatedToolCall.

### LlmStream Wrapper

The LlmStream struct wraps provider-specific streams into a unified interface using trait objects:

```
Pin<Box<dyn Stream<Item = LlmEvent> + Send>>
```

The Box provides heap allocation and ownership. The dyn enables dynamic dispatch—different providers can have different underlying stream types. The Pin ensures the stream can't be moved in memory (required for safe async usage). The Send bound allows the stream to be transferred between threads.

LlmStream implements the futures Stream trait, making it compatible with all standard stream combinators like filter, map, and take_while. It also provides a convenient next() async method for simple iteration:

```
let mut stream = client.complete_streaming(request).await?;
while let Some(event) = stream.next().await {
    match event {
        LlmEvent::TextDelta { content } => print!("{}", content),
        LlmEvent::Completed(response) => break,
        LlmEvent::Error(e) => return Err(e),
        _ => {}
    }
}
```

### ProxyLlmStream

The ProxyLlmStream in loom-llm-proxy parses the server's unified SSE format. This format uses "event: llm" for all events and discriminates by a type field in the JSON data:

```
event: llm
data: {"type":"text_delta","content":"Hello"}

event: llm
data: {"type":"tool_call_delta","call_id":"...","tool_name":"...","arguments_fragment":"..."}

event: llm
data: {"type":"completed","response":{...}}
```

The parser:
1. Buffers incoming bytes until it finds a complete event (double newline)
2. Extracts the "data:" line
3. Parses the JSON into LlmStreamEvent
4. Converts to LlmEvent
5. Yields the event to the consumer

This unified format means clients only need one parser instead of per-provider parsers. All provider-specific quirks are handled server-side.

### Wire Format Benefits

Having a unified wire format between server and clients provides:

**Single client implementation** - The CLI and web UI don't need provider-specific code.

**Provider isolation** - Adding a new provider only requires server changes.

**Simplified testing** - Mock the unified format without provider API dependencies.

**Forward compatibility** - New providers can be added server-side without client updates.

### Error Propagation

When an error occurs during streaming (network timeout, API error, invalid response), it's emitted as an LlmEvent::Error instead of causing immediate termination. This lets:

- The user see partial content before the error
- The application display a meaningful error message
- The agent's state machine handle the error gracefully (potentially retrying)

After emitting Error or Completed, the stream sets an internal finished flag and returns None on subsequent polls. This ensures clean termination.

### Retry Logic

Provider clients use the loom-http crate's retry mechanism for transient failures. The retry policy:

- Retries on network errors, timeouts, 429 rate limit, and 5xx server errors
- Does NOT retry on 4xx client errors (invalid request, auth failure)
- Uses exponential backoff with jitter to avoid thundering herds
- Respects Retry-After headers when present

For 429 rate limiting, the LlmError::RateLimited variant includes the retry_after_secs value from the API, which the agent can use for intelligent backoff.

## Why This Design?

**Provider Abstraction** - New LLM providers can be added without changing agent logic. Just implement LlmClient and register with LlmService.

**Security** - API keys never leave the server. Clients authenticate with user credentials, not provider credentials.

**Real-Time Feedback** - Streaming provides immediate feedback. You see text appearing, not a blank screen.

**Unified Interface** - One LlmEvent format works for all providers. The complexity is hidden in provider clients.

**Testability** - Mock implementations of LlmClient are trivial. Tests don't need real API keys or network access.

## Adding New Providers

To add a new LLM provider (e.g., Google Gemini):

1. Create loom-llm-gemini crate with GeminiClient implementing LlmClient.
2. Define API request/response types matching Gemini's format.
3. Implement conversion from LlmRequest to Gemini format and back to LlmResponse.
4. Create GeminiStream to parse Gemini's SSE format into LlmEvents.
5. Add Gemini API key environment variable to server config.
6. Update LlmService to instantiate GeminiClient if the key is present.
7. Add provider-specific methods: has_gemini(), complete_gemini(), complete_streaming_gemini().
8. Update server HTTP handlers to route /proxy/gemini/* to the Gemini methods.

Clients automatically get access to the new provider without any changes—they just use ProxyLlmClient::new(server_url, LlmProvider::Gemini).

## Implementation Location

**crates/loom-core/src/llm.rs** - LlmClient trait, LlmRequest, LlmResponse, LlmEvent, LlmStream
**crates/loom-llm-proxy/** - ProxyLlmClient and ProxyLlmStream implementations
**crates/loom-server-llm-service/** - LlmService that wraps provider clients
**crates/loom-server-llm-anthropic/** - AnthropicClient and Anthropic SSE parser
**crates/loom-server-llm-openai/** - OpenAIClient and OpenAI SSE parser
**crates/loom-server-api/src/llm_proxy.rs** - HTTP handlers for proxy endpoints
