---
summary: How OpenClaw handles long contexts: a complete guide
read_when:
  - You want to understand OpenClaw's long context handling strategy
  - You're experiencing issues with long conversations
  - You want to optimize context usage and costs
title: Long Context Handling
---

# How OpenClaw Handles Long Contexts

OpenClaw employs a multi-layered strategy to handle long conversations and large amounts of context, ensuring efficient operation even in long-running sessions. This document provides a comprehensive overview of all long context handling mechanisms.

## Core Challenge

All AI models have **context window limits** (maximum number of tokens they can process at once). Long conversations face these challenges:

- Conversation history grows continuously
- Tool call results accumulate
- System prompts and injected files consume space
- Exceeding limits causes request failures

OpenClaw solves these problems through **four layered mechanisms**.

## Four-Layer Long Context Handling

### 1. Context Window Management

**Purpose**: Monitor and limit context usage

**How it works**:
- Automatically detects model's context window size
- Real-time estimation of current session token usage
- Provides warning and blocking mechanisms to prevent overflow

**Configuration**:
```json5
{
  agents: {
    defaults: {
      contextTokens: 200000,  // Optional: cap context window
    }
  }
}
```

**Inspection commands**:
- `/status` - View current session token usage
- `/context list` - View context breakdown
- `/context detail` - Detailed context analysis

**Hard limits**:
- Minimum warning threshold: 32,000 tokens
- Minimum hard threshold: 16,000 tokens

See: [Context](/concepts/context)

### 2. Session Pruning

**Purpose**: Reduce tool result accumulation in context

**How it works**:
- Trims old tool results **in-memory only**
- **Does not modify** session history files on disk
- User and assistant messages are never pruned
- Protects the last N assistant messages (default 3)
- Tool results containing images are never pruned

**Two pruning modes**:

1. **Soft-trim**:
   - Keeps head and tail of tool results
   - Inserts `...` placeholder in the middle
   - Adds note with original size

2. **Hard-clear**:
   - Completely replaces tool result content
   - Uses brief placeholder text

**Trigger timing** (cache-ttl mode):
- When last Anthropic API call is older than TTL (default 5 minutes)
- Primarily used to optimize Anthropic prompt caching costs

**Default configuration**:
```json5
{
  agent: {
    contextPruning: {
      mode: "cache-ttl",
      ttl: "5m",
      keepLastAssistants: 3,
      softTrimRatio: 0.3,
      hardClearRatio: 0.5,
      minPrunableToolChars: 50000,
      softTrim: {
        maxChars: 4000,
        headChars: 1500,
        tailChars: 1500
      },
      hardClear: {
        enabled: true,
        placeholder: "[Old tool result content cleared]"
      }
    }
  }
}
```

**Benefits**:
- Reduces cache write costs
- Conversation history remains intact
- Lower re-caching costs

See: [Session Pruning](/concepts/session-pruning)

### 3. Auto-compaction

**Purpose**: Summarize long conversation history into compact summaries

**How it works**:
- Summarizes older conversation into a single summary message
- Keeps recent messages intact
- Summary is **persisted** to session's JSONL history file
- Future requests use "summary + recent messages"

**Trigger timing**:

1. **Overflow recovery**: When model returns context overflow error
2. **Threshold maintenance**: After successful reply when:
   ```
   contextTokens > contextWindow - reserveTokens
   ```

**Configuration**:
```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,      // Tokens reserved for output
    keepRecentTokens: 20000,    // Recent message tokens to keep
  }
}
```

**OpenClaw safety floor**:
- `reserveTokensFloor`: 20000 tokens (default)
- Ensures enough space for memory writes and other operations

**Manual compaction**:
```
/compact Focus on decisions and open questions
```

**Monitoring**:
- `/status` shows compaction count
- Verbose mode shows `🧹 Auto-compaction complete`

See: [Compaction](/concepts/compaction)

### 4. Memory Flush

**Purpose**: Save important information to disk before compaction

**How it works**:
- Triggers before reaching compaction threshold
- Runs a **silent AI turn**
- Prompts model to write important info to memory files
- Uses `NO_REPLY` to avoid user-visible output

**Trigger threshold**:
```
contextTokens > contextWindow - reserveTokensFloor - softThresholdTokens
```

**Default configuration**:
```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store.",
        }
      }
    }
  }
}
```

**Memory file layout**:
- `memory/YYYY-MM-DD.md` - Daily log (append-only)
- `MEMORY.md` - Long-term important memories

**When skipped**:
- Workspace is read-only (`workspaceAccess: "ro"` or `"none"`)
- Flush already executed in current compaction cycle

See: [Memory](/concepts/memory)

## Coordinated Workflow

Here's how the four mechanisms work together in a long conversation:

```
1. Session starts
   ↓
2. Normal conversation (context window management monitors)
   ↓
3. Reaches soft-trim threshold (30% of window)
   → Session pruning: Trim old tool results (in-memory only)
   ↓
4. Conversation continues
   ↓
5. Approaches compaction threshold (window - reserveTokensFloor - softThresholdTokens)
   → Memory flush: Silently save important info to disk
   ↓
6. Reaches compaction threshold
   → Auto-compaction: Summarize old history, persist summary
   ↓
7. Conversation continues (using summary + recent messages)
```

## Practical Usage Tips

### Daily Usage

1. **Monitor context usage**:
   ```
   /status
   ```
   Regularly check session token usage

2. **Check context breakdown**:
   ```
   /context list
   ```
   Understand what's consuming space

3. **Manual compaction**:
   ```
   /compact Remember to focus on X, Y, Z
   ```
   Manually compact at important points, preserving key info

4. **New session**:
   ```
   /new
   /reset
   ```
   Start fresh session for completely new topics

### Cost Optimization

1. **Use Anthropic caching**:
   - Enable `cache-ttl` pruning mode
   - Set appropriate heartbeat interval (30-60 minutes)

2. **Adjust retention parameters**:
   ```json5
   {
     compaction: {
       keepRecentTokens: 15000,  // Reduce retention
     },
     contextPruning: {
       keepLastAssistants: 2,    // Protect fewer messages
     }
   }
   ```

3. **Control injected file sizes**:
   ```json5
   {
     agents: {
       defaults: {
         bootstrapMaxChars: 15000  // Reduce per-file limit
       }
     }
   }
   ```

### Debugging Issues

1. **Frequent compaction**:
   - Check if model context window is too small
   - Higher `reserveTokens` may cause earlier compaction
   - Enable session pruning to reduce tool result accumulation

2. **Lost important information**:
   - Use `/compact` with instructions for manual compaction
   - Ensure memory flush is enabled
   - Proactively use `memory_search` and `memory_get` tools

3. **Compaction failures**:
   - Check if single message is too large (> 50% context window)
   - Review logs for specific errors
   - Consider switching to model with larger context window

## Choosing the Right Configuration

### Short sessions (< 10 turns)
- Default configuration works well
- No special optimization needed

### Medium sessions (10-50 turns)
- Enable session pruning (cache-ttl mode)
- Keep default compaction settings

### Long sessions (50+ turns)
- Enable all mechanisms
- Regular manual compaction
- Proactive use of memory files
- Consider segmenting with `/new` for new sessions

### Cost-sensitive scenarios
- Use Anthropic + cache-ttl pruning
- Reduce `keepRecentTokens`
- Limit `bootstrapMaxChars`
- Use heartbeat to keep cache alive

## Technical Implementation Details

### Context Estimation

OpenClaw uses character-to-token estimation:
```
Estimated tokens = character count / 4
```

This is an approximation; actual token count is determined by the model's tokenizer.

### Session Storage

- **Session store**: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **Session transcripts**: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

Session pruning only affects in-memory context; compaction modifies JSONL files.

### Compaction Algorithm

1. Split history into chunks (by token share)
2. Generate summary for each chunk
3. Merge summaries (if multiple)
4. Fallback strategy for oversized messages

Code location: `src/agents/compaction.ts`

### Pruning Algorithm

1. Find protection point (last N assistant messages)
2. Identify prunable tool results
3. Apply soft-trim (if over 30% window)
4. Apply hard-clear (if over 50% window)

Code location: `src/agents/pi-extensions/context-pruning/pruner.ts`

## Related Documentation

- [Context](/concepts/context) - Basic context concepts
- [Compaction](/concepts/compaction) - Detailed compaction explanation
- [Session Pruning](/concepts/session-pruning) - Pruning configuration and behavior
- [Memory](/concepts/memory) - Memory system and flush
- [Session](/concepts/session) - Session management basics
- [Session Management Deep Dive](/reference/session-management-compaction) - Technical deep dive

## FAQ

**Q: What's the difference between compaction and pruning?**

A:
- **Compaction**: Summarizes conversation history, **persists** to disk
- **Pruning**: Trims tool results, **in-memory only**, doesn't modify history files

**Q: Why do I see `🧹 Auto-compaction complete` messages?**

A: This indicates the session reached context limits and OpenClaw automatically compacted old history. This is normal behavior.

**Q: How can I reduce compaction frequency?**

A:
1. Use models with larger context windows
2. Enable session pruning
3. Reduce injected file sizes
4. Regularly use `/new` to start fresh sessions

**Q: Does compaction lose information?**

A: Compaction uses AI models to generate summaries, which preserve key information but may lose details. Use memory flush and manual compaction instructions to minimize information loss.

**Q: Can I disable auto-compaction?**

A: Technically yes by setting `compaction.enabled = false`, but not recommended. Long sessions without compaction will eventually fail due to context overflow.

**Q: What's the relationship between memory flush and compaction?**

A: Memory flush runs **before** compaction, giving the model a chance to write important information to permanent memory files, ensuring this information remains accessible after compaction.

**Q: How can I view compaction summary content?**

A: Compaction summaries are stored in the session's JSONL file (`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`). You can view the file directly or use session tools to read it.

**Q: When should I use `/compact` instead of waiting for auto-compaction?**

A:
- At important decision or summary points
- When conversation feels cluttered or repetitive
- When you want to add specific instructions to guide summary content
- Before switching to a new topic

## Summary

OpenClaw's long context handling is a carefully designed multi-layer system:

1. **Monitor** (Context window management) - Continuously track usage
2. **Optimize** (Session pruning) - Real-time reduction of tool result overhead
3. **Save** (Memory flush) - Persist important info before compaction
4. **Summarize** (Auto-compaction) - Compress history to stay within limits

These four mechanisms work together to ensure OpenClaw can support long-running, high-quality conversations while optimizing cost and performance.
