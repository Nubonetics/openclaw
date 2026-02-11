# OpenClaw — Implementation-Level Agent Framework Specification

## Purpose

This document is a precise, implementation-level specification of the OpenClaw AI agent framework. It documents the exact algorithms, data flows, type schemas, and edge-case handling as implemented in the TypeScript source code. The goal is to enable a code generator, given a YAML agent description, to produce source code that is **behaviorally identical** to the framework's implementation.

OpenClaw is a multi-channel AI gateway that routes user messages from messaging platforms (Signal, Telegram, Slack, Discord, WhatsApp, Web) through an embedded AI agent powered by the `@mariozechner/pi-coding-agent` SDK. It manages session persistence, tool execution, model fallback, streaming, compaction, and multi-agent orchestration.

---

## 1. Entry Point & Invocation Lifecycle

### 1.1 Public API Surface

The primary entry point for a user message is `runReplyAgent()` in `src/auto-reply/reply/agent-runner.ts:47`. This function is invoked by the channel-specific reply handler after message routing, session resolution, and queue evaluation.

**Signature:**
```typescript
async function runReplyAgent(params: {
  commandBody: string;              // The user's message text
  followupRun: FollowupRun;         // Contains run config, session info, prompt
  queueKey: string;                 // Queue identifier for followup deduplication
  resolvedQueue: QueueSettings;     // Queue mode and configuration
  shouldSteer: boolean;             // Whether to steer an active session
  shouldFollowup: boolean;          // Whether to enqueue as followup
  isActive: boolean;                // Whether a run is already active
  isStreaming: boolean;             // Whether streaming is active
  opts?: GetReplyOptions;           // Callbacks, abort signals, images
  typing: TypingController;         // Typing indicator controller
  sessionEntry?: SessionEntry;      // Current session metadata
  sessionStore?: Record<string, SessionEntry>;
  sessionKey?: string;              // Session routing key
  storePath?: string;               // Path to sessions.json
  defaultModel: string;             // Default model identifier
  agentCfgContextTokens?: number;   // Context window size override
  resolvedVerboseLevel: VerboseLevel;
  isNewSession: boolean;
  blockStreamingEnabled: boolean;
  blockReplyChunking?: { minChars; maxChars; breakPreference; flushOnParagraph };
  resolvedBlockStreamingBreak: "text_end" | "message_end";
  sessionCtx: TemplateContext;      // Templated message context (sender, channel, etc.)
  shouldInjectGroupIntro: boolean;
  typingMode: TypingMode;
}): Promise<ReplyPayload | ReplyPayload[] | undefined>
```

### 1.2 Invocation Context Construction

When a message arrives, the following objects are constructed before `runReplyAgent` is called:

1. **FollowupRun** (`src/auto-reply/reply/queue/types.ts`): Wraps the run configuration and prompt.
   - `run`: Contains `sessionId`, `sessionFile`, `config`, `provider`, `model`, `agentId`, `workspaceDir`, `thinkLevel`, `timeoutMs`, `extraSystemPrompt`, `ownerNumbers`, `skillsSnapshot`, `execOverrides`, `bashElevated`, `authProfileId`, `verboseLevel`, `reasoningLevel`
   - `prompt`: The user message text

2. **SessionEntry** (`src/config/sessions/types.ts`): Persistent session metadata stored in `sessions.json`.
   - Core fields: `sessionId` (UUID), `updatedAt` (epoch ms), `sessionFile` (transcript path)
   - Tracking: `systemSent`, `abortedLastRun`, `chatType`, `compactionCount`
   - Model config: `thinkingLevel`, `verboseLevel`, `reasoningLevel`, `elevatedLevel`, `providerOverride`, `modelOverride`, `authProfileOverride`
   - Token accounting: `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`
   - Queue config: `queueMode`, `queueDebounceMs`, `queueCap`, `queueDrop`
   - Delivery routing: `origin`, `deliveryContext`, `lastChannel`, `lastTo`, `lastAccountId`, `lastThreadId`

3. **TemplateContext** (`src/auto-reply/templating.ts`): Channel-specific message metadata.
   - `Provider`, `Surface`, `AccountId`, `SenderId`, `SenderName`, `SenderUsername`, `SenderE164`
   - `To`, `OriginatingTo`, `MessageSid`, `MessageSidFull`, `MessageThreadId`
   - `ChatType`, `GroupChannel`, `GroupSpace`, `GroupSubject`
   - `OriginatingChannel`

### 1.3 Pre-Run Decision Flow

Before the agent executes, `runReplyAgent` evaluates several conditions (lines 162–198):

1. **Steer check** (line 162): If `shouldSteer && isStreaming`, attempt to queue the message into the active embedded Pi session via `queueEmbeddedPiMessage()`. If steered and no followup needed, update `updatedAt` and return `undefined`.

2. **Queue check** (line 182): If `isActive && (shouldFollowup || resolvedQueue.mode === "steer")`, enqueue the followup run via `enqueueFollowupRun(queueKey, followupRun, resolvedQueue)`, update `updatedAt`, and return `undefined`.

3. **Memory flush** (line 202): Before running, call `runMemoryFlushIfNeeded()` which may trigger a memory consolidation pass if the session's compaction count exceeds a threshold.

### 1.4 Agent Selection Algorithm

The agent to run is determined by the session key, not by an explicit selection algorithm:

1. The `sessionKey` encodes the agent identity. For subagents, the key follows the pattern `<parentKey>::<agentId>`.
2. `resolveSessionAgentIds()` (`src/agents/agent-scope.ts`) extracts the agent ID:
   - `sessionAgentId`: Extracted from the session key via `isSubagentSessionKey()` / `normalizeAgentId()`
   - `defaultAgentId`: The root agent configured in `agents.defaults.agentId` or the global default
3. For subagent sessions, `promptMode` is set to `"minimal"` (reduced system prompt sections).
4. The `FollowupRun.run` object carries pre-resolved `provider`, `model`, `agentId`, and `agentDir`.

### 1.5 Session Initialization for a Run

Session preparation happens in `runEmbeddedAttempt()` (`src/agents/pi-embedded-runner/run/attempt.ts:140`):

1. **Workspace resolution** (line 159): `resolveRunWorkspaceDir()` determines the effective workspace directory, with fallback logic if the configured directory is unavailable.
2. **Sandbox context** (line 154): `resolveSandboxContext()` checks if Docker-based sandboxing is enabled.
3. **Session file repair** (line 410): `repairSessionFileIfNeeded()` fixes corrupted JSONL transcript files.
4. **SessionManager creation** (line 426): `SessionManager.open(params.sessionFile)` opens or creates the JSONL transcript.
5. **Session preparation** (line 433): `prepareSessionManagerForRun()` ensures the session has proper headers and state.
6. **Agent session creation** (line 478): `createAgentSession()` from the Pi SDK creates the agent with tools, model, and settings.
7. **System prompt override** (line 490): The generated system prompt is applied via `applySystemPromptOverrideToSession()`, which replaces both `agent.setSystemPrompt()` and the internal `_rebuildSystemPrompt` callback.

---

## 2. The Core Loop

### 2.1 Outer Loop Structure

OpenClaw does **not** implement its own ReAct loop. The core loop is delegated to the `@mariozechner/pi-coding-agent` SDK via `activeSession.prompt(effectivePrompt)` (line 820-823 in `attempt.ts`). The SDK internally implements the reason-act cycle:

```
User prompt → LLM call → [tool calls → tool results → LLM call]* → final response
```

OpenClaw wraps this with two nested retry loops:

**Loop 1: Auth Profile Rotation** (`run.ts:357-384`)
```typescript
while (profileIndex < profileCandidates.length) {
  // Skip profiles in cooldown
  // Apply API key
  // Break on success
}
```

**Loop 2: Model Retry with Compaction** (`run.ts:392-860`)
```typescript
while (true) {
  attemptedThinking.add(thinkLevel);
  const attempt = await runEmbeddedAttempt({...});

  // Check for context overflow → auto-compact → retry
  // Check for auth failure → rotate profile → retry
  // Check for thinking level incompatibility → downgrade → retry
  // On success → break
}
```

### 2.2 Per-Step Execution (SDK-Level)

Within the SDK's `activeSession.prompt()`, each step:
1. Sends the conversation (system prompt + history + user message) to the LLM
2. Receives a streaming response
3. If the response contains tool calls, executes them and loops
4. If the response is a final text response, terminates

OpenClaw observes these steps via the **subscription pattern** (`subscribeEmbeddedPiSession()` in `src/agents/pi-embedded-subscribe.ts:31`). The subscription registers an event handler that processes:
- `message_start` → Reset assistant state, signal typing
- `message_update` → Accumulate text deltas, emit streaming events
- `message_end` → Finalize assistant text, emit block replies
- `tool_execution_start` → Flush block replies, emit tool start events
- `tool_execution_update` → Emit tool progress
- `tool_execution_end` → Record tool metadata, commit messaging tool sends
- `agent_start` → Emit lifecycle start event
- `agent_end` → Drain remaining buffers, resolve compaction waits
- `auto_compaction_start` → Mark compaction in-flight
- `auto_compaction_end` → Handle retry or resolve

### 2.3 Termination Conditions

The agent loop terminates when:

1. **Final response**: The SDK's agent produces a text response with no tool calls (normal completion).
2. **Abort signal** (line 573): External `AbortSignal` triggers `abortRun()` → `runAbortController.abort()` → `activeSession.abort()`.
3. **Timeout** (line 669): `setTimeout` fires after `params.timeoutMs`, calling `abortRun(true)`.
4. **Context overflow** (lines 491-598 in `run.ts`): Detected via `isContextOverflowError()` on prompt errors or assistant error messages. Triggers auto-compaction (up to `MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3`), then tool result truncation, then returns error payload.
5. **Role ordering conflict** (line 490): Detected via regex `/incorrect role information|roles must alternate/i`. Resets session.
6. **Session corruption** (line 507 in `agent-runner-execution.ts`): Gemini-specific function call ordering error. Deletes transcript and session entry.
7. **Auth exhaustion**: All auth profiles fail or are in cooldown. Throws `FailoverError`.
8. **Compaction failure** (line 500 in `run.ts`): `isCompactionFailureError()` returns true. Resets session with new UUID.

### 2.4 Maximum Iteration Guard

There is no explicit maximum iteration count within OpenClaw itself. The guard is the **timeout** (`params.timeoutMs`), typically configured per-agent. The SDK may have its own internal limits. OpenClaw's auto-compaction has a hard limit of `MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3` retries (`run.ts:386`).

---

## 3. Preprocessing Pipeline

### 3.1 System Prompt Assembly

The system prompt is assembled in `buildEmbeddedSystemPrompt()` → `buildAgentSystemPrompt()` (`src/agents/system-prompt.ts:164`). The assembly order is:

1. **Identity line**: `"You are a personal assistant running inside OpenClaw."`
2. **Tooling section**: Lists available tools with summaries, filtered by policy. Tool order is hardcoded (`toolOrder` array, line 248). Extra tools (not in `toolOrder`) are appended alphabetically.
3. **Tool Call Style section**: Instructions on when to narrate tool calls.
4. **Safety section**: Anti-goal-seeking, human oversight, no self-modification.
5. **CLI Quick Reference**: OpenClaw subcommand guidance.
6. **Skills section** (if not minimal): Scan `<available_skills>`, read exactly one SKILL.md.
7. **Memory section** (if not minimal and memory tools available): Instructions for `memory_search`/`memory_get`.
8. **Self-Update section** (if gateway tool available and not minimal): Config/update guidance.
9. **Model Aliases section** (if not minimal): Preferred model alias names.
10. **Time hint**: `"If you need the current date...run session_status"` (if timezone available).
11. **Workspace section**: Working directory path and workspace notes.
12. **Documentation section** (if not minimal): Docs path, URLs.
13. **Sandbox section** (if sandboxed): Docker sandbox details, workspace access, elevated exec.
14. **Runtime Info section**: Host, OS, arch, Node version, model, provider, channel, capabilities.
15. **User Identity section** (if not minimal): Owner numbers.
16. **Reply Tags section** (if not minimal): `[[reply_to_current]]` and `[[reply_to:<id>]]` syntax.
17. **Messaging section** (if not minimal): Channel routing, message tool guidance, inline buttons.
18. **Voice section** (if not minimal): TTS hints.
19. **Heartbeat section** (if default agent): Heartbeat prompt.
20. **Reasoning hint** (if reasoning tag provider): `<think>...</think>` and `<final>...</final>` format instructions.
21. **Extra system prompt**: User-configured `extraSystemPrompt` appended verbatim.
22. **Context files**: Bootstrap files (e.g., `OPENCLAW.md`) injected as XML-tagged content blocks.

**Prompt modes** (`PromptMode`):
- `"full"` (default): All sections included
- `"minimal"`: Reduced sections (Tooling, Workspace, Runtime only). Used for subagents.
- `"none"`: Returns only `"You are a personal assistant running inside OpenClaw."`

### 3.2 Tool Registration

Tools are created in `createOpenClawCodingTools()` (`src/agents/pi-tools.ts:115`). The pipeline:

1. **Resolve effective tool policy** (`resolveEffectiveToolPolicy()`): Merges global, agent-scoped, and provider-scoped policies.
2. **Resolve group policy** (`resolveGroupToolPolicy()`): Channel/group-level restrictions.
3. **Create core tools**: `read`, `write`, `edit`, `apply_patch`, `grep`, `find`, `ls`, `exec`, `process` from `@mariozechner/pi-coding-agent` and custom implementations.
4. **Create OpenClaw tools** (`createOpenClawTools()`): `web_search`, `web_fetch`, `browser`, `canvas`, `nodes`, `cron`, `message`, `gateway`, `agents_list`, `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `session_status`, `image`, `memory_search`, `memory_get`.
5. **Create channel-specific tools** (`listChannelAgentTools()`): Plugin-provided tools.
6. **Apply sandbox wrapping**: If sandbox enabled, wrap `read`/`write`/`edit` with sandboxed variants.
7. **Apply abort signal wrapping** (`wrapToolWithAbortSignal()`): Each tool checks `abortSignal` before execution.
8. **Apply before-tool-call hook** (`wrapToolWithBeforeToolCallHook()`): Runs plugin hooks.
9. **Filter by policy** (`filterToolsByPolicy()`): Remove tools not allowed by the effective policy chain.
10. **Apply owner-only restrictions** (`applyOwnerOnlyToolPolicy()`): Gate tools to owner senders.
11. **Schema normalization** (`normalizeToolParameters()`): Ensure parameter schemas are model-compatible.
12. **Gemini sanitization** (`cleanToolSchemaForGemini()`): Provider-specific schema adjustments.
13. **Split for SDK** (`splitSdkTools()`): Separates tools into `builtInTools` (SDK-native) and `customTools` (OpenClaw-specific).

### 3.3 Conversation History Preparation

History is prepared in `runEmbeddedAttempt()` (lines 542-566):

1. **sanitizeSessionHistory()**: Normalizes message history for the target model API. Handles provider-specific quirks (Gemini, Anthropic).
2. **validateGeminiTurns()**: If transcript policy requires it, ensures Gemini-compatible turn ordering.
3. **validateAnthropicTurns()**: If transcript policy requires it, ensures Anthropic-compatible role alternation.
4. **limitHistoryTurns()**: Applies DM-specific history turn limits from config (`getDmHistoryLimitFromSessionKey()`).
5. **replaceMessages()**: Commits the sanitized, validated, limited messages back to the agent.

### 3.4 Hook-Based Prompt Modification

Before the prompt is sent to the SDK (lines 726-748):

```typescript
if (hookRunner?.hasHooks("before_agent_start")) {
  const hookResult = await hookRunner.runBeforeAgentStart(
    { prompt: params.prompt, messages: activeSession.messages },
    { agentId: hookAgentId, sessionKey, workspaceDir, messageProvider }
  );
  if (hookResult?.prependContext) {
    effectivePrompt = `${hookResult.prependContext}\n\n${params.prompt}`;
  }
}
```

Hooks can prepend context to the prompt but cannot modify messages or tools.

### 3.5 Orphaned User Message Repair

Before prompting (lines 758-771):

```typescript
const leafEntry = sessionManager.getLeafEntry();
if (leafEntry?.type === "message" && leafEntry.message.role === "user") {
  // Branch away from the orphaned user message to prevent consecutive user turns
  if (leafEntry.parentId) {
    sessionManager.branch(leafEntry.parentId);
  } else {
    sessionManager.resetLeaf();
  }
  activeSession.agent.replaceMessages(sessionContext.messages);
}
```

### 3.6 Image Detection and Injection

Before prompting (lines 778-798):

1. `detectAndLoadPromptImages()` scans the prompt text for file paths matching image extensions.
2. For vision-capable models, images are loaded as base64 `ImageContent` objects.
3. `injectHistoryImagesIntoMessages()` scans conversation history and re-injects images into their original message positions (for follow-up questions about earlier images).
4. If history messages were mutated, `replaceMessages()` persists the changes.

---

## 4. Contents Construction Algorithm

### 4.1 Session Transcript Structure

Session transcripts are stored as JSONL files managed by `SessionManager` from the Pi SDK. Each entry has:
- `type`: `"message"` | `"header"` | `"summary"` | `"tool_use"` | `"tool_result"`
- `message`: The `AgentMessage` (role, content, etc.)
- `parentId`: For tree-structured branching
- Entries form a directed acyclic graph (DAG) where `buildSessionContext()` linearizes the active branch.

### 4.2 Message Filtering and Sanitization

`sanitizeSessionHistory()` (`src/agents/pi-embedded-runner/google.ts`) applies:
1. Provider-specific content normalization (e.g., stripping unsupported content types)
2. Tool use/result pairing validation
3. Role ordering validation

`validateGeminiTurns()` ensures:
- No consecutive function call turns without intervening model responses
- Tool results are properly paired with tool use entries

`validateAnthropicTurns()` ensures:
- Strict user/assistant role alternation
- No consecutive same-role messages

### 4.3 History Limiting

`limitHistoryTurns()` (`src/agents/pi-embedded-runner/history.ts`):
- Takes a `maxTurns` parameter from config (via `getDmHistoryLimitFromSessionKey()`)
- Keeps the most recent N turns (a turn = user message + assistant response)
- Always preserves the first message (summary) if one exists

### 4.4 Branch-Based Context

The SessionManager supports branching:
- `sessionManager.branch(parentId)` sets the active branch point
- `sessionManager.buildSessionContext()` linearizes messages along the active branch
- `sessionManager.getLeafEntry()` returns the most recent entry on the current branch
- `sessionManager.resetLeaf()` discards the current leaf entry

### 4.5 Compaction and Summarization

When context exceeds limits, `compactEmbeddedPiSessionDirect()` (`src/agents/pi-embedded-runner/compact.ts`) is triggered:

1. **Token estimation**: `estimateMessagesTokens()` calculates total tokens.
2. **Adaptive chunk ratio**: `computeAdaptiveChunkRatio()` adjusts based on average message size:
   - `BASE_CHUNK_RATIO = 0.4`
   - `MIN_CHUNK_RATIO = 0.15`
   - If avgMessage > 10% of context, ratio decreases
3. **Chunking**: `chunkMessagesByMaxTokens()` splits messages into chunks no larger than `maxChunkTokens`.
4. **Summarization**: Each chunk is summarized via `generateSummary()` from the Pi SDK with instructions to preserve decisions, TODOs, constraints.
5. **Merge**: If multiple summaries, they are merged with `MERGE_SUMMARIES_INSTRUCTIONS`.
6. **Oversized check**: `isOversizedForSummary()` flags messages > 50% of context window.

### 4.6 Tool Result Truncation

When compaction alone cannot resolve overflow (`src/agents/pi-embedded-runner/tool-result-truncation.ts`):
- `sessionLikelyHasOversizedToolResults()` scans for tool results that individually exceed a fraction of the context window
- `truncateOversizedToolResultsInSession()` rewrites oversized tool results in the session file with truncated versions

---

## 5. LLM Call Mechanics

### 5.1 Method Chain

The LLM invocation chain is:

```
runReplyAgent()
  → runAgentTurnWithFallback()
    → runWithModelFallback()
      → runEmbeddedPiAgent()
        → runEmbeddedAttempt()
          → activeSession.prompt(effectivePrompt)
            → [Pi SDK internal: agent.streamFn(messages, tools, system)]
```

### 5.2 Model Fallback

`runWithModelFallback()` (`src/agents/model-fallback.ts:209`) implements cascading model fallback:

1. **Candidate resolution** (`resolveFallbackCandidates()`):
   - Start with the requested `provider/model`
   - Append models from `agents.defaults.model.fallbacks[]` config
   - Append the globally configured primary model
   - Deduplicate by `provider/model` key
   - Enforce allowlist if configured

2. **Execution loop**:
   ```typescript
   for (let i = 0; i < candidates.length; i++) {
     // Skip if all auth profiles for this provider are in cooldown
     try {
       const result = await params.run(candidate.provider, candidate.model);
       return { result, provider, model, attempts };
     } catch (err) {
       if (shouldRethrowAbort(err)) throw err;       // AbortError → rethrow
       if (!isFailoverError(normalized)) throw err;   // Non-failover → rethrow
       attempts.push({ provider, model, error, reason, status, code });
       await params.onError?.({...});                 // Notify caller
     }
   }
   ```

3. **Abort handling**: `isFallbackAbortError()` returns true only for `AbortError` name (not message-based), preventing timeout masking.

### 5.3 Auth Profile Rotation

Within `runEmbeddedPiAgent()` (`src/agents/pi-embedded-runner/run.ts`):

1. **Profile ordering** (`resolveAuthProfileOrder()`): Ordered list of auth profiles for the provider.
2. **Cooldown check** (`isProfileInCooldown()`): Profiles recently failed are skipped.
3. **Rotation** (`advanceAuthProfile()`): On failure, advance to next non-cooldown profile. Resets thinking level and attempted-thinking set.
4. **Locked profiles**: If `authProfileIdSource === "user"`, the profile cannot be rotated.
5. **Failure marking** (`markAuthProfileFailure()`): Records failure reason and timestamp.
6. **Success marking** (`markAuthProfileGood()` + `markAuthProfileUsed()`): Clears cooldown and records usage.

### 5.4 Streaming Mode

The SDK uses `streamSimple` from `@mariozechner/pi-ai` (line 518 in `attempt.ts`):
```typescript
activeSession.agent.streamFn = streamSimple;
```

Streaming is the default mode. The subscription handler (`subscribeEmbeddedPiSession()`) processes streaming events:
- `text_delta`: Appended to `deltaBuffer`, fed to block chunker
- `text_start`: May contain initial content
- `text_end`: May resend full content (provider quirk); only the suffix beyond `deltaBuffer` is appended
- Block chunking splits streamed text into delivery-sized chunks

Non-streaming (batch) mode is not explicitly supported; the SDK always streams.

### 5.5 LLM Call Count Tracking

Token usage is tracked via `UsageAccumulator` (`run.ts:77-135`):
```typescript
type UsageAccumulator = {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  total: number;
};
```

After each attempt, usage is merged: `mergeUsageIntoAccumulator(usageAccumulator, attempt.attemptUsage)`.

Usage is also tracked per-subscription via `recordAssistantUsage()` in the subscribe handler, which normalizes and accumulates across multiple assistant messages within a single run.

### 5.6 Error Handling During LLM Calls

Errors are classified and handled in the retry loop (`run.ts:473-598`):

| Error Type | Detection | Action |
|---|---|---|
| Context overflow | `isContextOverflowError(text)` | Auto-compact (3 attempts) → tool truncation → return error payload |
| Compaction failure | `isCompactionFailureError(text)` | Reset session (new UUID) → return error payload |
| Role ordering | `/incorrect role information\|roles must alternate/i` | Return error payload with kind `"role_ordering"` |
| Image size | `parseImageSizeError(text)` | Return friendly error with max size hint |
| Auth failure | `isAuthAssistantError()` | Mark profile failed → rotate → retry |
| Rate limit | `isRateLimitAssistantError()` | Mark profile failed → rotate → retry |
| Billing | `isBillingAssistantError()` | Mark profile failed → rotate → retry |
| Timeout | Timer expires / `isTimeoutErrorMessage()` | Mark profile (timeout) → rotate → retry |
| Thinking level | `pickFallbackThinkingLevel()` | Downgrade thinking → retry |
| Gemini corruption | `/function call turn comes immediately after/i` | Delete transcript, delete session entry → return error |

---

## 6. Tool Dispatch & Execution

### 6.1 Tool Creation Pipeline

Tools are created in `createOpenClawCodingTools()` (`src/agents/pi-tools.ts:115`):

```typescript
function createOpenClawCodingTools(options): AnyAgentTool[] {
  // 1. Resolve effective tool policy (global → agent → provider → group)
  const { globalPolicy, agentPolicy, profile, ... } = resolveEffectiveToolPolicy({...});
  const groupPolicy = resolveGroupToolPolicy({...});

  // 2. Create base tools
  const coreTools = [read, write, edit, apply_patch, grep, find, ls, exec, process];
  const openclawTools = createOpenClawTools({...});
  const channelTools = listChannelAgentTools({...});

  // 3. Apply wrappers
  tools = tools.map(t => wrapToolWithAbortSignal(t, abortSignal));
  tools = tools.map(t => wrapToolWithBeforeToolCallHook(t));

  // 4. Filter by policy
  tools = filterToolsByPolicy(tools, effectivePolicy);
  tools = applyOwnerOnlyToolPolicy(tools, senderIsOwner);

  // 5. Normalize schemas
  tools = tools.map(t => normalizeToolParameters(t));
  if (isGeminiProvider) tools = tools.map(t => cleanToolSchemaForGemini(t));

  return tools;
}
```

### 6.2 Tool Split for SDK

`splitSdkTools()` (`src/agents/pi-embedded-runner/tool-split.ts`) categorizes tools:
- **builtInTools**: Tools recognized by the Pi SDK (`read`, `write`, `edit`, `exec`, etc.)
- **customTools**: All other tools (OpenClaw-specific, channel, plugin tools)

This split is required because the SDK handles built-in tools differently (e.g., file editing with its own execution logic).

### 6.3 Tool Execution Lifecycle (Subscription Events)

Tool execution is observed via the subscription event handler:

**1. `tool_execution_start`** (`src/agents/pi-embedded-subscribe.handlers.tools.ts:39`):
```typescript
async function handleToolExecutionStart(ctx, evt) {
  ctx.flushBlockReplyBuffer();           // Flush pending text before tool
  ctx.params.onBlockReplyFlush?.();      // External flush callback

  const toolName = normalizeToolName(evt.toolName);
  const meta = inferToolMetaFromArgs(toolName, args);  // e.g., file path for read
  ctx.state.toolMetaById.set(toolCallId, meta);

  emitAgentEvent({ runId, stream: "tool", data: { phase: "start", name, toolCallId, args } });
  ctx.params.onAgentEvent?.({ stream: "tool", data: { phase: "start", name, toolCallId } });

  if (shouldEmitToolResult() && onToolResult) {
    ctx.emitToolSummary(toolName, meta);  // e.g., "🔧 read · /path/to/file"
  }

  // Track pending messaging tool sends
  if (isMessagingTool(toolName) && isMessagingToolSendAction(toolName, args)) {
    ctx.state.pendingMessagingTexts.set(toolCallId, text);
    ctx.state.pendingMessagingTargets.set(toolCallId, target);
  }
}
```

**2. `tool_execution_update`** (`handlers.tools.ts:116`):
```typescript
function handleToolExecutionUpdate(ctx, evt) {
  const sanitized = sanitizeToolResult(evt.partialResult);
  emitAgentEvent({ runId, stream: "tool", data: { phase: "update", name, toolCallId, partialResult } });
  ctx.params.onAgentEvent?.({ stream: "tool", data: { phase: "update", ... } });
}
```

**3. `tool_execution_end`** (`handlers.tools.ts:148`):
```typescript
function handleToolExecutionEnd(ctx, evt) {
  const isToolError = evt.isError || isToolResultError(result);
  ctx.state.toolMetas.push({ toolName, meta });

  if (isToolError) {
    ctx.state.lastToolError = { toolName, meta, error: extractToolErrorMessage(result) };
  }

  // Commit or discard pending messaging tool texts
  const pendingText = ctx.state.pendingMessagingTexts.get(toolCallId);
  if (pendingText && !isToolError) {
    ctx.state.messagingToolSentTexts.push(pendingText);
    ctx.state.messagingToolSentTextsNormalized.push(normalizeTextForComparison(pendingText));
  }

  emitAgentEvent({ runId, stream: "tool", data: { phase: "result", name, toolCallId, isError, result } });

  if (shouldEmitToolOutput()) {
    ctx.emitToolOutput(toolName, meta, extractToolResultText(result));
  }
}
```

### 6.4 Tool Abort Wrapping

`wrapToolWithAbortSignal()` (`src/agents/pi-tools.abort.ts`):
- Wraps every tool's `run()` method
- Before execution: checks if `abortSignal.aborted` → throws `AbortError`
- After execution: checks again
- This ensures long-running tools can be interrupted

### 6.5 Messaging Tool Duplicate Detection

To prevent duplicate messages (agent sends via tool then also responds with same text):

1. **Pending tracking**: On `tool_execution_start`, if the tool is a messaging tool with send action, the text is stored in `pendingMessagingTexts`.
2. **Commit on success**: On `tool_execution_end`, if no error, text moves to `messagingToolSentTexts` and its normalized form to `messagingToolSentTextsNormalized`.
3. **Block reply suppression**: In `emitBlockChunk()`, the chunk is compared against committed messaging texts via `isMessagingToolDuplicateNormalized()`. Matches are suppressed.
4. **Cap**: `MAX_MESSAGING_SENT_TEXTS = 200` and `MAX_MESSAGING_SENT_TARGETS = 200`. Oldest entries are trimmed when exceeded.

---

## 7. Event System & Session Persistence

### 7.1 Agent Event Schema

Defined in `src/infra/agent-events.ts`:

```typescript
type AgentEventPayload = {
  runId: string;        // UUID identifying this run
  seq: number;          // Strictly monotonic per runId (auto-incremented)
  stream: AgentEventStream;  // "lifecycle" | "tool" | "assistant" | "error" | string
  ts: number;           // Date.now() at emission
  data: Record<string, unknown>;  // Event-specific payload
  sessionKey?: string;  // Resolved from run context
};

type AgentRunContext = {
  sessionKey?: string;
  verboseLevel?: VerboseLevel;
  isHeartbeat?: boolean;
};
```

### 7.2 Event Emission

`emitAgentEvent()` (line 57):
```typescript
function emitAgentEvent(event: Omit<AgentEventPayload, "seq" | "ts">) {
  const nextSeq = (seqByRun.get(event.runId) ?? 0) + 1;
  seqByRun.set(event.runId, nextSeq);
  const context = runContextById.get(event.runId);
  const sessionKey = event.sessionKey ?? context?.sessionKey;
  const enriched = { ...event, sessionKey, seq: nextSeq, ts: Date.now() };
  for (const listener of listeners) {
    try { listener(enriched); } catch { /* ignore */ }
  }
}
```

Key properties:
- **Per-run sequence numbers**: `seqByRun` map ensures strictly monotonic ordering per `runId`
- **Timestamp enrichment**: `ts: Date.now()` added automatically
- **Session key resolution**: Falls back to registered run context
- **Listener isolation**: Errors in listeners are swallowed

### 7.3 Event Streams

| Stream | Phases/Data | Source |
|---|---|---|
| `lifecycle` | `phase: "start"` (with `startedAt`), `"end"` (with `endedAt`), `"error"` | `handleAgentStart`, `handleAgentEnd`, CLI runner |
| `tool` | `phase: "start"` (name, args), `"update"` (partialResult), `"result"` (isError, result) | `handleToolExecution*` handlers |
| `assistant` | `text` (full), `delta` (incremental), `mediaUrls` | `handleMessageUpdate`, `handleMessageEnd` |
| `compaction` | `phase: "start"`, `"end"` (with `willRetry`) | `handleAutoCompaction*` handlers |

### 7.4 Session Persistence

**Session Store** (`src/config/sessions/store.ts`):
- Format: JSON5 file at `~/.openclaw/sessions/sessions.json`
- Structure: `Record<string, SessionEntry>` keyed by session key
- Concurrency: File-based lock (`{storePath}.lock`) with 10s timeout, 25ms poll, 30s stale eviction
- Cache: TTL of 45s (`OPENCLAW_SESSION_CACHE_TTL_MS`), invalidated on write, mtime-checked

**Session Transcripts**:
- Format: JSONL files at `~/.openclaw/sessions/{sessionId}.jsonl`
- Managed by `SessionManager` from the Pi SDK
- Header includes: version, sessionId, timestamp, cwd
- Entries: Messages, tool uses, tool results, summaries (from compaction)

**Store Operations**:
```typescript
// Atomic read-modify-write with file lock
async function updateSessionStore<T>(
  storePath: string,
  mutator: (store: Record<string, SessionEntry>) => Promise<T> | T,
): Promise<T>

// Single-entry update
async function updateSessionStoreEntry(params: {
  storePath, sessionKey,
  update: (entry: SessionEntry) => Promise<Partial<SessionEntry> | null>,
}): Promise<SessionEntry | null>
```

**Maintenance**:
- `pruneStaleEntries()`: Removes entries older than 30 days (configurable)
- `capEntryCount()`: Keeps 500 most recent entries (configurable)
- `rotateSessionFile()`: Renames store to `.bak.{timestamp}` when > 10MB, keeps 3 backups

### 7.5 Diagnostic Events

`src/infra/diagnostic-events.ts` provides a separate event system for operational telemetry:
- `DiagnosticUsageEvent`: Model usage (tokens, costs, duration)
- `DiagnosticWebhook*Event`: Webhook reception and processing
- `DiagnosticMessage*Event`: Message queue and processing
- `DiagnosticSessionStateEvent`: Session state transitions (idle/processing/waiting)
- `DiagnosticSessionStuckEvent`: Stuck session detection with age
- `DiagnosticLane*Event`: Queue lane management
- `DiagnosticRunAttemptEvent`: Run attempt tracking
- `DiagnosticHeartbeatEvent`: Aggregate webhook/session stats

### 7.6 System Events

`src/infra/system-events.ts`: Lightweight, in-memory, per-session event queue:
- `MAX_EVENTS = 20` per session
- Duplicate suppression (consecutive text duplicates skipped)
- Not persisted to disk (ephemeral by design)
- Drained before each prompt

---

## 8. Agent Transfer Mechanism

### 8.1 Multi-Agent Architecture

OpenClaw supports multi-agent orchestration through **session-based agent spawning**, not transfer-based delegation. Agents are identified by `agentId` and communicate via session keys.

### 8.2 Session Key Structure

Session keys encode agent identity (`src/routing/session-key.ts`):
- Direct message: `<channel>:<accountId>:<senderId>`
- Group: `<channel>:<accountId>:group:<groupId>`
- Subagent: `<parentSessionKey>::<agentId>`
- `isSubagentSessionKey()` checks for the `::` separator
- `normalizeAgentId()` extracts the agent portion

### 8.3 Agent Scope Resolution

`resolveSessionAgentIds()` (`src/agents/agent-scope.ts`):
```typescript
function resolveSessionAgentIds(params: {
  sessionKey?: string;
  config?: OpenClawConfig;
}): { defaultAgentId: string; sessionAgentId: string }
```
- `defaultAgentId`: From `agents.defaults.agentId` config or global default
- `sessionAgentId`: Extracted from session key if subagent, otherwise `defaultAgentId`

### 8.4 Sub-Agent Spawning

Via the `sessions_spawn` tool:
1. A new session key is created: `<parentSessionKey>::<agentId>`
2. A new `sessionId` (UUID) is generated
3. The sub-agent's system prompt uses `promptMode: "minimal"`
4. The sub-agent runs independently with its own transcript
5. Communication happens via `sessions_send` tool

### 8.5 Agent-Specific Configuration

`resolveAgentModelFallbacksOverride()` (`src/agents/agent-scope.ts`):
- Agents can have their own model fallback chains
- Per-agent tool policy via `agents.<agentId>.tools.policy`
- Per-agent model overrides via session entry `modelOverride`/`providerOverride`

---

## 9. Postprocessing Pipeline

### 9.1 Response Text Processing

After the SDK run completes, response processing occurs in multiple stages:

**Stage 1: Subscription-level processing** (during the run):
- `stripBlockTags()`: Removes `<think>...</think>` content (stateful across chunks)
- `<final>...</final>` enforcement: If `enforceFinalTag` is true, only text inside `<final>` blocks is emitted
- `stripDowngradedToolCallText()`: Removes `[Tool Call: ...]` text artifacts
- `parseReplyDirectives()`: Extracts `[[reply_to_current]]`, `[[media:url]]`, `[[audio_as_voice]]` directives
- Messaging tool duplicate suppression: Skips text already sent via messaging tools

**Stage 2: Payload construction** (`buildEmbeddedRunPayloads()` in `src/agents/pi-embedded-runner/run/payloads.ts`):
- Collects `assistantTexts[]` accumulated during the run
- Collects `toolMetas[]` for tool usage summaries
- Formats error payloads from `lastAssistant.errorMessage`
- Applies `verboseLevel` and `reasoningLevel` to include/exclude tool details and reasoning

**Stage 3: Final payload assembly** (`buildReplyPayloads()` in `src/auto-reply/reply/agent-runner-payloads.ts`):
- Heartbeat token stripping: `stripHeartbeatToken()` removes `HEARTBEAT_OK` tokens
- Silent reply detection: `isSilentReplyText()` checks for `SILENT_REPLY_TOKEN`
- Text sanitization: `sanitizeUserFacingText()` removes internal markers
- Reply-to mode filtering: `applyReplyToMode()` applies channel threading
- Reply tag application: `applyReplyTagsToPayload()` converts `[[reply_to:id]]` to payload fields
- Messaging tool deduplication: Filters payloads whose text matches `messagingToolSentTexts`

**Stage 4: Response decoration** (in `runReplyAgent()`, lines 495-514):
- If auto-compaction completed: Prepend `"🧹 Auto-compaction complete"` (if verbose)
- If new session: Prepend `"🧭 New session: {sessionId}"` (if verbose)
- If response usage enabled: Append usage line (tokens, cost)

### 9.2 Block Streaming

Block streaming delivers partial responses during the run:

1. **Block chunker** (`EmbeddedBlockChunker`): Accumulates text and emits chunks based on:
   - `minChars` / `maxChars` thresholds
   - `breakPreference`: `"paragraph"` | `"newline"` | `"sentence"`
   - `flushOnParagraph`: Force emit on paragraph break

2. **Block reply pipeline** (`createBlockReplyPipeline()`): Manages timed flushing and coalescing:
   - `timeoutMs`: Maximum time to hold before sending
   - Coalescing: Combines nearby small chunks
   - Audio-as-voice buffering

3. **Break modes**:
   - `"text_end"`: Emit block on `text_end` event (within assistant message)
   - `"message_end"`: Emit block on `message_end` event (full message completion)

---

## 10. Callback System

### 10.1 Run-Level Callbacks

Defined in `GetReplyOptions` and passed through the call chain:

| Callback | Signature | When it fires |
|---|---|---|
| `onPartialReply` | `(payload: { text?, mediaUrls? }) → void \| Promise<void>` | On each streamed text delta (after think/final stripping) |
| `onBlockReply` | `(payload: { text?, mediaUrls?, audioAsVoice?, replyToId?, replyToTag?, replyToCurrent? }) → void \| Promise<void>` | On each chunked block of text ready for delivery |
| `onBlockReplyFlush` | `() → void \| Promise<void>` | Before tool execution (flush pending blocks) |
| `onReasoningStream` | `(payload: { text?, mediaUrls? }) → void \| Promise<void>` | On reasoning/thinking text deltas |
| `onToolResult` | `(payload: { text?, mediaUrls? }) → void \| Promise<void>` | On tool summary/output (if verbose) |
| `onAssistantMessageStart` | `() → void \| Promise<void>` | When assistant message_start event fires |
| `onAgentEvent` | `(evt: { stream, data }) → void` | On every agent event (lifecycle, tool, assistant) |
| `onAgentRunStart` | `(runId: string) → void` | When a run begins (before model selection) |
| `onModelSelected` | `(info: { provider, model, thinkLevel }) → void` | After model fallback resolution |
| `abortSignal` | `AbortSignal` | External abort mechanism |

### 10.2 Hook System

The hook system (`src/plugins/hook-runner-global.ts`) provides two extension points:

**`before_agent_start`** (line 726 in `attempt.ts`):
```typescript
interface BeforeAgentStartInput {
  prompt: string;
  messages: AgentMessage[];
}
interface BeforeAgentStartContext {
  agentId: string;
  sessionKey?: string;
  workspaceDir: string;
  messageProvider?: string;
}
interface BeforeAgentStartResult {
  prependContext?: string;  // Prepended to the prompt
}
```
- Runs synchronously before prompting
- Can inject context but not modify messages or tools
- Errors are caught and logged (non-fatal)

**`agent_end`** (line 854 in `attempt.ts`):
```typescript
interface AgentEndInput {
  messages: AgentMessage[];
  success: boolean;
  error?: string;
  durationMs: number;
}
interface AgentEndContext {
  agentId: string;
  sessionKey?: string;
  workspaceDir: string;
  messageProvider?: string;
}
```
- Runs fire-and-forget (not awaited)
- For analytics, logging, post-processing
- Errors are caught and logged (non-fatal)

### 10.3 Agent Event Listeners

Global event listeners registered via `onAgentEvent()`:
```typescript
function onAgentEvent(listener: (evt: AgentEventPayload) => void): () => void
```
- Returns unsubscribe function
- Listeners receive enriched events (with `seq`, `ts`, `sessionKey`)
- Errors in listeners are silently swallowed
- Used by gateway WebSocket server, diagnostics, and UI

### 10.4 Typing Signal Callbacks

`createTypingSignaler()` wraps the typing controller with mode-aware signaling:
- `signalRunStart()`: Indicate run has begun
- `signalTextDelta(text)`: Indicate text is being generated
- `signalToolStart()`: Indicate tool execution is beginning
- `signalMessageStart()`: Indicate assistant message is starting
- `signalReasoningDelta()`: Indicate reasoning text is being generated

### 10.5 Session Transcript Listeners

`onSessionTranscriptUpdate()` (`src/sessions/transcript-events.ts`):
```typescript
function onSessionTranscriptUpdate(listener: SessionTranscriptListener): () => void
function emitSessionTranscriptUpdate(sessionFile: string): void
```
- Fires when a transcript file is modified
- Used by the UI to refresh conversation display

---

## 11. Multi-Agent Orchestration

### 11.1 Session-Based Agent Identity

Each agent has its own session, identified by the session key structure:
- Root agent: `<channel>:<accountId>:<senderId>`
- Sub-agent: `<parentKey>::<agentId>`

The `resolveSessionAgentIds()` function determines which agent configuration to apply:
```typescript
const isSubagent = isSubagentSessionKey(sessionKey);
const sessionAgentId = isSubagent ? extractAgentId(sessionKey) : defaultAgentId;
```

### 11.2 Sub-Agent System Prompt

Sub-agents receive a reduced system prompt (`promptMode: "minimal"`):
- Includes: Tooling, Workspace, Runtime Info sections
- Excludes: Skills, Memory, Self-Update, Model Aliases, Reply Tags, Messaging, Voice, Heartbeat, Documentation, Safety
- Tool policy can be further restricted via `resolveSubagentToolPolicy()`

### 11.3 Inter-Agent Communication

Agents communicate via tools:
- `sessions_send(sessionKey, message)`: Send a message to another session
- `sessions_list()`: List available sessions
- `sessions_history(sessionKey)`: Retrieve conversation history
- `sessions_spawn(agentId)`: Create a new sub-agent session

### 11.4 Queue-Based Message Processing

Messages are processed through a lane-based queue system:

1. **Session lane** (`resolveSessionLane()`): Per-session sequential processing
2. **Global lane** (`resolveGlobalLane()`): Cross-session rate limiting
3. **Nesting**: `enqueueSession(() => enqueueGlobal(async () => { ... }))` ensures both session-level serialization and global concurrency control

### 11.5 Followup Queue

When a run is active and a new message arrives (`src/auto-reply/reply/queue/`):
- `QueueMode`: `"steer"` | `"followup"` | `"collect"` | `"steer-backlog"` | `"steer+backlog"` | `"queue"` | `"interrupt"`
- `"steer"`: Inject into active session via `queueEmbeddedPiMessage()`
- `"followup"`: Enqueue and drain after current run completes
- `"queue"`: Standard FIFO queue
- `QueueDropPolicy`: `"old"` | `"new"` | `"summarize"` when cap exceeded
- `queueCap`: Maximum queue depth
- `queueDebounceMs`: Debounce interval

### 11.6 Session Reset and Recovery

Session reset (`resetSession()` in `agent-runner.ts:235`):
```typescript
const resetSession = async ({ failureLabel, buildLogMessage, cleanupTranscripts }) => {
  const nextSessionId = crypto.randomUUID();
  const nextEntry = {
    ...prevEntry,
    sessionId: nextSessionId,
    updatedAt: Date.now(),
    systemSent: false,
    abortedLastRun: false,
  };
  nextEntry.sessionFile = resolveSessionTranscriptPath(nextSessionId, agentId, threadId);
  // Update store atomically
  await updateSessionStore(storePath, (store) => { store[sessionKey] = nextEntry; });
  followupRun.run.sessionId = nextSessionId;
  followupRun.run.sessionFile = nextEntry.sessionFile;
  // Optionally delete old transcript files
};
```

Triggers:
- Compaction failure → `resetSessionAfterCompactionFailure()`
- Role ordering conflict → `resetSessionAfterRoleOrderingConflict()`
- Gemini session corruption → Delete transcript + session entry

---

## Appendix A: Complete Request Processor Chain

The full processing chain from user message to LLM call:

```
1. Channel webhook received (Telegram, Signal, Slack, etc.)
2. Message routing → session key resolution
3. Queue evaluation (steer / followup / process)
4. runReplyAgent()
   a. Steer check → queueEmbeddedPiMessage() if active
   b. Queue check → enqueueFollowupRun() if should followup
   c. Memory flush check → runMemoryFlushIfNeeded()
   d. runAgentTurnWithFallback()
      i. Generate runId (UUID)
      ii. Register agent run context
      iii. runWithModelFallback()
         - Resolve fallback candidates
         - For each candidate:
           iv. runEmbeddedPiAgent()
               - Resolve workspace, provider, model, auth
               - Context window guard evaluation
               - Auth profile resolution and key application
               - While (retry loop):
                 v. runEmbeddedAttempt()
                    1. Resolve sandbox context
                    2. Load workspace skill entries
                    3. Resolve bootstrap context files
                    4. Create tools (createOpenClawCodingTools)
                    5. Split tools (splitSdkTools)
                    6. Build system prompt (buildEmbeddedSystemPrompt)
                    7. Acquire session write lock
                    8. Repair session file if needed
                    9. Open SessionManager
                    10. Prepare SessionManager for run
                    11. Create agent session (SDK)
                    12. Apply system prompt override
                    13. Sanitize session history
                    14. Validate turns (Gemini/Anthropic)
                    15. Limit history turns
                    16. Replace messages
                    17. Subscribe to session events
                    18. Set active embedded run
                    19. Run before_agent_start hooks
                    20. Repair orphaned user messages
                    21. Detect and load prompt images
                    22. Inject history images
                    23. activeSession.prompt(effectivePrompt)
                        → SDK internal ReAct loop
                        → Streaming events → subscription handlers
                    24. Wait for compaction retry
                    25. Run agent_end hooks (fire-and-forget)
                    26. Unsubscribe, cleanup, release lock
                    27. Return EmbeddedRunAttemptResult
5. Post-run processing
   a. Usage persistence (persistSessionUsageUpdate)
   b. Payload construction (buildReplyPayloads)
   c. Response decoration (compaction notice, session hint, usage line)
   d. Followup scheduling (finalizeWithFollowup)
```

---

## Appendix B: Event Schema Reference

### AgentEventPayload
```typescript
{
  runId: string;          // UUID
  seq: number;            // Per-run monotonic counter (starts at 1)
  stream: "lifecycle" | "tool" | "assistant" | "error" | "compaction" | string;
  ts: number;             // Date.now() at emission
  data: Record<string, unknown>;
  sessionKey?: string;    // From run context or explicit
}
```

### Lifecycle Events
```typescript
// Start
{ phase: "start", startedAt: number }
// End
{ phase: "end", endedAt: number }
// Error (startedAt included for duration calculation)
{ phase: "error", startedAt: number, endedAt: number, error: string }
```

### Tool Events
```typescript
// Start
{ phase: "start", name: string, toolCallId: string, args: Record<string, unknown> }
// Update
{ phase: "update", name: string, toolCallId: string, partialResult: unknown }
// Result
{ phase: "result", name: string, toolCallId: string, meta?: string, isError: boolean, result: unknown }
```

### Assistant Events
```typescript
{ text: string, delta: string, mediaUrls?: string[] }
```

### Compaction Events
```typescript
// Start
{ phase: "start" }
// End
{ phase: "end", willRetry: boolean }
```

### SessionEntry (Session Store)
```typescript
{
  sessionId: string;        // UUID
  updatedAt: number;        // Epoch ms
  sessionFile?: string;     // Transcript JSONL path
  spawnedBy?: string;       // Parent session key (sub-agents)
  systemSent?: boolean;     // Whether system intro was sent
  abortedLastRun?: boolean;
  chatType?: "dm" | "group" | "thread";
  thinkingLevel?: string;
  verboseLevel?: string;
  reasoningLevel?: string;
  elevatedLevel?: string;
  ttsAuto?: "off" | "on" | "voice";
  execHost?: string;
  execSecurity?: string;
  execAsk?: string;
  execNode?: string;
  responseUsage?: "on" | "off" | "tokens" | "full";
  providerOverride?: string;
  modelOverride?: string;
  authProfileOverride?: string;
  authProfileOverrideSource?: "auto" | "user";
  groupActivation?: "mention" | "always";
  groupActivationNeedsSystemIntro?: boolean;
  sendPolicy?: "allow" | "deny";
  queueMode?: "steer" | "followup" | "collect" | "steer-backlog" | "steer+backlog" | "queue" | "interrupt";
  queueDebounceMs?: number;
  queueCap?: number;
  queueDrop?: "old" | "new" | "summarize";
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  contextTokens?: number;
  compactionCount?: number;
  lastHeartbeatText?: string;
  lastHeartbeatSentAt?: number;
  memoryFlushAt?: number;
  label?: string;
  displayName?: string;
  channel?: string;
  groupId?: string;
  subject?: string;
  origin?: SessionOrigin;
  deliveryContext?: DeliveryContext;
  lastChannel?: SessionChannelId;
  lastTo?: string;
  lastAccountId?: string;
  lastThreadId?: string | number;
  skillsSnapshot?: SessionSkillSnapshot;
  systemPromptReport?: SessionSystemPromptReport;
}
```

---

## Appendix C: LlmRequest Schema Reference

### EmbeddedRunAttemptParams (the "LLM Request")
```typescript
{
  // Session identity
  sessionId: string;
  sessionKey?: string;
  agentId?: string;

  // Channel context
  messageChannel?: string;
  messageProvider?: string;
  agentAccountId?: string;
  messageTo?: string;
  messageThreadId?: string | number;
  groupId?: string | null;
  groupChannel?: string | null;
  groupSpace?: string | null;
  spawnedBy?: string | null;
  senderId?: string | null;
  senderName?: string | null;
  senderUsername?: string | null;
  senderE164?: string | null;
  senderIsOwner?: boolean;
  currentChannelId?: string;
  currentThreadTs?: string;
  replyToMode?: "off" | "first" | "all";
  hasRepliedRef?: { value: boolean };

  // Session files
  sessionFile: string;    // JSONL transcript path
  workspaceDir: string;   // Agent workspace directory
  agentDir?: string;      // Agent configuration directory

  // Configuration
  config?: OpenClawConfig;
  skillsSnapshot?: SkillSnapshot;

  // Prompt
  prompt: string;         // User message text
  images?: ImageContent[]; // Vision content
  clientTools?: ClientToolDefinition[];
  disableTools?: boolean;

  // Model
  provider: string;       // e.g., "anthropic", "openai", "google"
  modelId: string;        // e.g., "claude-sonnet-4-5-20250929"
  model: Model<Api>;      // Resolved model object with capabilities
  authStorage: AuthStorage;
  modelRegistry: ModelRegistry;

  // Execution params
  thinkLevel: "off" | "low" | "medium" | "high";
  verboseLevel?: "off" | "on" | "full";
  reasoningLevel?: "off" | "on" | "stream";
  toolResultFormat?: "markdown" | "plain";
  execOverrides?: { host?; security?; ask?; node? };
  bashElevated?: ExecElevatedDefaults;
  timeoutMs: number;
  runId: string;          // UUID for this run
  abortSignal?: AbortSignal;

  // Streaming callbacks
  shouldEmitToolResult?: () => boolean;
  shouldEmitToolOutput?: () => boolean;
  onPartialReply?: (payload) => void | Promise<void>;
  onAssistantMessageStart?: () => void | Promise<void>;
  onBlockReply?: (payload) => void | Promise<void>;
  onBlockReplyFlush?: () => void | Promise<void>;
  blockReplyBreak?: "text_end" | "message_end";
  blockReplyChunking?: BlockReplyChunking;
  onReasoningStream?: (payload) => void | Promise<void>;
  onToolResult?: (payload) => void | Promise<void>;
  onAgentEvent?: (evt) => void;

  // System prompt
  extraSystemPrompt?: string;
  streamParams?: AgentStreamParams;
  ownerNumbers?: string[];
  enforceFinalTag?: boolean;
}
```

### EmbeddedRunAttemptResult (the "LLM Response")
```typescript
{
  aborted: boolean;
  timedOut: boolean;
  promptError: unknown;           // Error during prompt submission
  sessionIdUsed: string;          // Actual session ID used
  systemPromptReport?: SessionSystemPromptReport;
  messagesSnapshot: AgentMessage[];  // Full conversation after run
  assistantTexts: string[];       // Cleaned response texts
  toolMetas: Array<{ toolName: string; meta?: string }>;
  lastAssistant: AssistantMessage | undefined;
  lastToolError?: { toolName: string; meta?: string; error?: string };
  didSendViaMessagingTool: boolean;
  messagingToolSentTexts: string[];
  messagingToolSentTargets: MessagingToolSend[];
  cloudCodeAssistFormatError: boolean;
  attemptUsage?: NormalizedUsage;
  compactionCount?: number;
  clientToolCall?: { name: string; params: Record<string, unknown> };
}
```

### EmbeddedPiRunResult (the "Final Result")
```typescript
{
  payloads?: Array<{
    text?: string;
    mediaUrl?: string;
    mediaUrls?: string[];
    replyToId?: string;
    isError?: boolean;
  }>;
  meta: {
    durationMs: number;
    agentMeta?: {
      sessionId: string;
      provider: string;
      model: string;
      compactionCount?: number;
      usage?: { input?; output?; cacheRead?; cacheWrite?; total? };
    };
    aborted?: boolean;
    systemPromptReport?: SessionSystemPromptReport;
    error?: {
      kind: "context_overflow" | "compaction_failure" | "role_ordering" | "image_size";
      message: string;
    };
    stopReason?: string;
    pendingToolCalls?: Array<{ id; name; arguments }>;
  };
  didSendViaMessagingTool?: boolean;
  messagingToolSentTexts?: string[];
  messagingToolSentTargets?: MessagingToolSend[];
}
```

### NormalizedUsage
```typescript
{
  input?: number;       // Input tokens
  output?: number;      // Output tokens
  cacheRead?: number;   // Cached input tokens read
  cacheWrite?: number;  // Cached input tokens written
  total?: number;       // Total tokens (derived if not provided)
}
```

### SubscribeEmbeddedPiSessionParams
```typescript
{
  session: AgentSession;
  runId: string;
  verboseLevel?: "off" | "on" | "full";
  reasoningMode?: "off" | "on" | "stream";
  toolResultFormat?: "markdown" | "plain";
  shouldEmitToolResult?: () => boolean;
  shouldEmitToolOutput?: () => boolean;
  onToolResult?: (payload) => void | Promise<void>;
  onReasoningStream?: (payload) => void | Promise<void>;
  onBlockReply?: (payload) => void | Promise<void>;
  onBlockReplyFlush?: () => void | Promise<void>;
  blockReplyBreak?: "text_end" | "message_end";
  blockReplyChunking?: BlockReplyChunking;
  onPartialReply?: (payload) => void | Promise<void>;
  onAssistantMessageStart?: () => void | Promise<void>;
  onAgentEvent?: (evt) => void | Promise<void>;
  enforceFinalTag?: boolean;
}
```

### EmbeddedPiSubscribeState (Internal Subscription State)
```typescript
{
  assistantTexts: string[];
  toolMetas: Array<{ toolName?; meta? }>;
  toolMetaById: Map<string, string | undefined>;
  toolSummaryById: Set<string>;
  lastToolError?: ToolErrorSummary;
  blockReplyBreak: "text_end" | "message_end";
  reasoningMode: "off" | "on" | "stream";
  includeReasoning: boolean;
  shouldEmitPartialReplies: boolean;
  streamReasoning: boolean;
  deltaBuffer: string;            // Accumulated streamed text
  blockBuffer: string;            // Accumulated block text
  blockState: { thinking; final; inlineCode };  // Stateful tag tracking
  partialBlockState: { thinking; final; inlineCode };
  lastStreamedAssistant?: string;
  lastStreamedAssistantCleaned?: string;
  emittedAssistantUpdate: boolean;
  lastBlockReplyText?: string;
  assistantMessageIndex: number;
  lastAssistantTextMessageIndex: number;
  lastAssistantTextNormalized?: string;
  lastAssistantTextTrimmed?: string;
  assistantTextBaseline: number;
  suppressBlockChunks: boolean;
  lastReasoningSent?: string;
  compactionInFlight: boolean;
  pendingCompactionRetry: number;
  compactionRetryResolve?: () => void;
  compactionRetryPromise: Promise<void> | null;
  messagingToolSentTexts: string[];
  messagingToolSentTextsNormalized: string[];
  messagingToolSentTargets: MessagingToolSend[];
  pendingMessagingTexts: Map<string, string>;
  pendingMessagingTargets: Map<string, MessagingToolSend>;
}
```
