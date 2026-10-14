# PowerContext gateway plugin

Automatic context recall and completed-conversation capture for clients behind
Magpie. Requires a Magpie build supporting the proposed async gateway hooks in
[yetone/magpie#2118](https://github.com/yetone/magpie/issues/2118). This is not
compatible with stable versions that only expose synchronous middleware.

```mermaid
sequenceDiagram
    participant Client
    participant Magpie
    participant Plugin as PowerContext hook
    participant PC as Standalone PowerContext
    participant Model
    Client->>Magpie: Request
    Magpie->>Plugin: onRequest
    Plugin->>PC: Resolve Scope, prepare context, latest Handoff
    PC-->>Plugin: Untrusted reference text
    Plugin-->>Magpie: Context-enriched request
    Magpie->>Model: One inference request
    Model-->>Client: Live response through Magpie
    Magpie--)Plugin: Completed response notification
    Plugin--)PC: Complete conversation Source
    PC->>PC: Background Memory extraction
    Note over PC: Future sessions can recall generated Memory
```

## Installation and connection

Install and operate PowerContext separately, following its
[official installation walkthrough](https://powercontext.oceanbase.io/en/docs/get-started/quickstart/).
The plugin does not install dependencies, download executables, launch a Server,
or manage its lifecycle. The current reference service is PowerContext 1.3.0.
Use the Server configuration wizard to configure storage, authentication and
background generation. Keep the Server running under your own process manager.
Follow the Server installation/configuration steps only; the separate Codex or
Claude hook setup in that walkthrough is unnecessary for gateway recall.
Create a Scope using PowerContext and configure its ID below.

This package is unpublished and the Magpie hook API is not released. With an
explicitly compatible development build, install from the local checkout:

~~~sh
magpie plugin add "/absolute/path/to/packages/powercontext"
magpie plugin options "/absolute/path/to/packages/powercontext" '{"serverUrl":"http://127.0.0.1:8000","scopeId":"YOUR_SCOPE","authorizationEnv":"POWERCONTEXT_AUTHORIZATION","gatewayTimeoutMs":5000}'
~~~

Provide POWERCONTEXT_AUTHORIZATION=Bearer <server token> in Magpie's existing
protected environment configuration. Do not put credentials in Git or client
model prompts. For remote Servers, use HTTPS. Installation and options use the
same Magpie plugin controls as other packages; installing this plugin does not
start the independent Server. Automatic recall/capture requires no client MCP
or skill.

## Behavior and options

- `serverUrl`: standalone Server base URL (default `http://127.0.0.1:8000`).
- `authorizationEnv`: bearer authorization variable (default `POWERCONTEXT_AUTHORIZATION`).
  `authorization` is also accepted in protected local plugin options.
- `scopeId`: explicit Scope. If absent, resolve an existing Magpie session or
  workspace binding. A missing binding fails open; the plugin does not create
  scopes or grant a model permission to change them.
- `maxBytes`: recall budget, default 8000 bytes.
- `gatewayTimeoutMs`: Magpie's bounded async hook deadline, default 5000 ms.
- `capture`: enable completed-conversation capture, default true.
- `idleCaptureMs`: default ten minutes; range 1000–86400000 ms. A later user
  turn confirming a retained answer can trigger capture earlier.
- `allowInsecureHttp`: explicit opt-in for non-loopback HTTP; prefer HTTPS.

Chat Completions, Responses, Anthropic Messages and Gemini requests retain their
protocol. Context is escaped and enclosed as untrusted reference text in the
latest user message, preserving prior context blocks for prompt caching. Latest
committed Handoff hints accompany recall when available. Credentials are not
included in model input. Timeout, invalid context, unauthorized access and a
service outage keep inference available.

Capture stores the complete conversation available to the request: user and
assistant messages, plus client-owned tool calls and results when supplied by
the client. It excludes injected reference blocks and private authorization.
Responses `previous_response_id` history is recovered only for capture when
observed by this host; missing history does not modify model input or pretend to
be a complete transcript. Failed, truncated and incomplete replies are skipped.

The gateway completion notification and plugin's idle capture timer are
best-effort. Process shutdown can lose pending capture; this is not durable or
exactly-once delivery. Source identifiers/digests provide stable content
identity, but uncertain writes must not be blindly retried.

PowerContext's configured background inference turns Sources into Memory. It
needs its own generation model and credentials; the plugin does not reuse the
client's model authorization. Configure generation through the independent
Server's documented setup.

## Memory tools

This plugin neither declares nor executes memory tools. If an Agent needs
explicit search or remember tools, configure PowerContext's existing MCP server
through the client's Library MCP integration. The client owns tool execution and
subsequent model calls. Automatic recall/capture requires no MCP or skill.

## Validation

Run `bun test packages/powercontext` from the repository root. Tests use fake
HTTP responses and do not install or start PowerContext. Independent real-service
acceptance is maintained outside this repository, with results posted in the PR.
Passing those checks against a development Magpie build does not establish
compatibility with a released host; adoption must wait for #2118 and a compatible
Magpie version.
