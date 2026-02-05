---
summary: OpenClaw 如何处理长上下文：完整指南
read_when:
  - 你想了解 OpenClaw 的长上下文处理策略
  - 你在使用长对话时遇到问题
  - 你想优化上下文使用和成本
title: 长上下文处理
---

# OpenClaw 如何处理长上下文

OpenClaw 采用多层策略来处理长对话和大量上下文，确保即使在长时间运行的会话中也能保持高效运行。本文档综合介绍所有长上下文处理机制。

## 核心挑战

所有 AI 模型都有**上下文窗口限制**（可以同时处理的最大 token 数量）。长对话会面临以下挑战：

- 对话历史不断增长
- 工具调用结果累积
- 系统提示词和注入文件占用空间
- 超过限制会导致请求失败

OpenClaw 通过**四层机制**解决这些问题。

## 四层长上下文处理机制

### 1. 上下文窗口管理

**作用**：监控和限制上下文使用

**工作原理**：
- 自动检测模型的上下文窗口大小
- 实时估算当前会话使用的 token 数量
- 提供警告和阻止机制防止超限

**配置项**：
```json5
{
  agents: {
    defaults: {
      contextTokens: 200000,  // 可选：限制上下文窗口上限
    }
  }
}
```

**检查命令**：
- `/status` - 查看当前会话的 token 使用情况
- `/context list` - 查看上下文分解
- `/context detail` - 详细的上下文分析

**硬性限制**：
- 最小警告阈值：32,000 tokens
- 最小硬性阈值：16,000 tokens

详见：[上下文](/concepts/context)

### 2. 会话修剪（Session Pruning）

**作用**：减少工具结果在上下文中的累积

**工作原理**：
- 仅在**内存中**裁剪旧的工具结果
- **不修改**磁盘上的会话历史文件
- 用户和助手消息永远不会被修剪
- 保护最近的 N 个助手消息（默认 3 个）
- 包含图片的工具结果不会被修剪

**两种修剪模式**：

1. **软修剪**（Soft-trim）：
   - 保留工具结果的头部和尾部
   - 中间插入 `...` 占位符
   - 添加原始大小说明

2. **硬清除**（Hard-clear）：
   - 完全替换工具结果内容
   - 用简短占位符替代

**触发时机**（cache-ttl 模式）：
- 当距离上次 Anthropic API 调用超过 TTL 时间（默认 5 分钟）
- 主要用于优化 Anthropic 提示词缓存成本

**默认配置**：
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

**优点**：
- 减少缓存写入成本
- 对话历史保持完整
- 降低重新缓存的成本

详见：[会话修剪](/concepts/session-pruning)

### 3. 自动压缩（Auto-compaction）

**作用**：将长对话历史总结为紧凑摘要

**工作原理**：
- 将较旧的对话总结成一条摘要消息
- 保留最近的消息不变
- 摘要**持久化**到会话的 JSONL 历史文件中
- 后续请求使用"摘要 + 最近消息"

**触发时机**：

1. **溢出恢复**：模型返回上下文溢出错误时
2. **阈值维护**：成功回复后检测到：
   ```
   contextTokens > contextWindow - reserveTokens
   ```

**配置项**：
```json5
{
  compaction: {
    enabled: true,
    reserveTokens: 16384,      // 为输出预留的 token
    keepRecentTokens: 20000,    // 保留的最近消息 token 数
  }
}
```

**OpenClaw 安全底线**：
- `reserveTokensFloor`: 20000 tokens（默认）
- 确保有足够空间进行记忆写入等操作

**手动压缩**：
```
/compact Focus on decisions and open questions
```

**监控**：
- `/status` 显示压缩次数
- 详细模式显示 `🧹 Auto-compaction complete`

详见：[压缩](/concepts/compaction)

### 4. 记忆刷写（Memory Flush）

**作用**：在压缩前保存重要信息到磁盘

**工作原理**：
- 在接近压缩阈值前触发
- 运行**静默的 AI 轮次**
- 提示模型将重要信息写入记忆文件
- 使用 `NO_REPLY` 避免用户看到中间输出

**触发阈值**：
```
contextTokens > contextWindow - reserveTokensFloor - softThresholdTokens
```

**默认配置**：
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

**记忆文件布局**：
- `memory/YYYY-MM-DD.md` - 每日日志（追加）
- `MEMORY.md` - 长期重要记忆

**何时跳过**：
- 工作区为只读模式（`workspaceAccess: "ro"` 或 `"none"`）
- 同一压缩周期已执行过刷写

详见：[记忆](/concepts/memory)

## 机制协同工作流程

以下是四层机制在长对话中的协同工作流程：

```
1. 会话开始
   ↓
2. 正常对话（上下文窗口管理监控）
   ↓
3. 达到软修剪阈值（30% 窗口）
   → 会话修剪：裁剪旧工具结果（仅内存）
   ↓
4. 继续对话
   ↓
5. 接近压缩阈值（窗口 - reserveTokensFloor - softThresholdTokens）
   → 记忆刷写：静默保存重要信息到磁盘
   ↓
6. 达到压缩阈值
   → 自动压缩：总结旧历史，持久化摘要
   ↓
7. 继续对话（使用摘要 + 最近消息）
```

## 实际使用建议

### 日常使用

1. **监控上下文使用**：
   ```
   /status
   ```
   定期检查会话的 token 使用情况

2. **检查上下文分解**：
   ```
   /context list
   ```
   了解哪些内容占用空间

3. **手动压缩**：
   ```
   /compact Remember to focus on X, Y, Z
   ```
   在重要节点手动压缩，保留关键信息

4. **新会话**：
   ```
   /new
   /reset
   ```
   在全新主题时开始新会话

### 优化成本

1. **使用 Anthropic 缓存**：
   - 启用 `cache-ttl` 修剪模式
   - 设置合适的心跳间隔（30-60分钟）

2. **调整保留参数**：
   ```json5
   {
     compaction: {
       keepRecentTokens: 15000,  // 减少保留量
     },
     contextPruning: {
       keepLastAssistants: 2,    // 保护更少消息
     }
   }
   ```

3. **控制注入文件大小**：
   ```json5
   {
     agents: {
       defaults: {
         bootstrapMaxChars: 15000  // 减少每文件限制
       }
     }
   }
   ```

### 调试问题

1. **频繁压缩**：
   - 检查模型上下文窗口是否太小
   - 增加 `reserveTokens` 可能导致更早压缩
   - 启用会话修剪减少工具结果累积

2. **重要信息丢失**：
   - 使用 `/compact` 带指令手动压缩
   - 确保记忆刷写已启用
   - 主动使用 `memory_search` 和 `memory_get` 工具

3. **压缩失败**：
   - 检查单条消息是否过大（> 50% 上下文窗口）
   - 查看日志了解具体错误
   - 考虑切换到更大上下文窗口的模型

## 不同模式的选择

### 短会话（< 10 轮对话）
- 默认配置即可
- 不需要特殊优化

### 中等会话（10-50 轮）
- 启用会话修剪（cache-ttl 模式）
- 保持默认压缩设置

### 长会话（50+ 轮）
- 启用所有机制
- 定期手动压缩
- 主动使用记忆文件
- 考虑分段使用 `/new` 开始新会话

### 成本敏感场景
- 使用 Anthropic + cache-ttl 修剪
- 减少 `keepRecentTokens`
- 限制 `bootstrapMaxChars`
- 使用心跳保持缓存活跃

## 技术实现细节

### 上下文估算

OpenClaw 使用字符到 token 的估算：
```
估算 tokens = 字符数 / 4
```

这是近似值，实际 token 数由模型的 tokenizer 决定。

### 会话存储

- **会话存储**：`~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **会话记录**：`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

会话修剪只影响内存中的上下文，压缩会修改 JSONL 文件。

### 压缩算法

1. 将历史分块（基于 token 份额）
2. 对每个块生成摘要
3. 合并摘要（如果有多个）
4. 处理超大消息的降级策略

代码位置：`src/agents/compaction.ts`

### 修剪算法

1. 找到保护点（最后 N 个助手消息）
2. 识别可修剪的工具结果
3. 应用软修剪（如果超过 30% 窗口）
4. 应用硬清除（如果超过 50% 窗口）

代码位置：`src/agents/pi-extensions/context-pruning/pruner.ts`

## 相关文档

- [上下文](/concepts/context) - 上下文的基本概念
- [压缩](/concepts/compaction) - 压缩的详细说明
- [会话修剪](/concepts/session-pruning) - 修剪的配置和工作原理
- [记忆](/concepts/memory) - 记忆系统和刷写
- [会话](/concepts/session) - 会话管理基础
- [会话管理深入](/reference/session-management-compaction) - 技术深入解析

## 常见问题

**Q: 压缩和修剪有什么区别？**

A:
- **压缩**：总结对话历史，**持久化**到磁盘
- **修剪**：裁剪工具结果，**仅在内存中**，不修改历史文件

**Q: 为什么看到 `🧹 Auto-compaction complete` 消息？**

A: 这表示会话达到了上下文限制，OpenClaw 自动压缩了旧历史。这是正常行为。

**Q: 如何减少压缩频率？**

A:
1. 使用更大上下文窗口的模型
2. 启用会话修剪
3. 减少注入文件大小
4. 定期使用 `/new` 开始新会话

**Q: 压缩会丢失信息吗？**

A: 压缩使用 AI 模型生成摘要，会保留关键信息但可能丢失细节。使用记忆刷写和手动压缩指令可以减少信息丢失。

**Q: 能关闭自动压缩吗？**

A: 技术上可以设置 `compaction.enabled = false`，但不推荐。没有压缩的长会话最终会因上下文溢出而失败。

**Q: 记忆刷写和压缩的关系？**

A: 记忆刷写在压缩**之前**运行，给模型机会将重要信息写入永久记忆文件，确保压缩后仍能访问这些信息。

**Q: 如何查看压缩摘要内容？**

A: 压缩摘要存储在会话的 JSONL 文件中（`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`），可以直接查看文件或使用会话工具读取。

**Q: 什么时候应该使用 `/compact` 而不是等待自动压缩？**

A:
- 在重要决策或总结点
- 感觉对话变得混乱或重复时
- 想要添加特定指令来指导摘要内容时
- 准备切换到新主题前

## 总结

OpenClaw 的长上下文处理是一个精心设计的多层系统：

1. **监控**（上下文窗口管理）- 持续跟踪使用情况
2. **优化**（会话修剪）- 实时减少工具结果开销
3. **保存**（记忆刷写）- 在压缩前持久化重要信息
4. **总结**（自动压缩）- 压缩历史保持在限制内

这四个机制协同工作，确保 OpenClaw 能够支持长时间、高质量的对话，同时优化成本和性能。
