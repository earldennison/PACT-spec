# PACT Selector Examples

This document provides comprehensive examples of PACT context selectors for real-world use cases.

See [04 – Context Queries](../spec/04-queries.md) for the formal specification.

---

## Basic Patterns

### Selecting by Region
```javascript
// All system context
ctx.select("^sys .block")

// Everything in sealed sequence
ctx.select("^seq")

// Only active head content
ctx.select("^ah .block")

// All content blocks across entire tree
ctx.select("^root .block")

// All nodes with created_at_ns beyond a threshold (numeric comparison)
ctx.select("^root .block[created_at_ns>1724670000000000000]")
```

### Selecting by Type
```javascript
// All message turns
ctx.select("^seq .seg")

// All message containers
ctx.select("^seq .cont")

// All content blocks
ctx.select("^root .block")

// Namespaced user types and kinds
ctx.select("^seq .block:summary")            // type token (nodeType="block", kind="summary")
ctx.select("^root .block[kind='tool_call']") // kind is an attribute
```

### Selecting by Tag
```javascript
// All user messages
ctx.select("^root .block +user")

// All assistant responses
ctx.select("^root .block +assistant")

// System messages only
ctx.select("^sys .block +system")

// Tool calls and results
ctx.select("^root .block +tool")
```

---

## Positional Selectors

### Recent Turns
```javascript
// Most recent historical turn
ctx.select("( d1 )")

// Last 3 historical turns
ctx.select("( d1..d3 )")

// Last turn's user message
ctx.select("( d1, .block ) +user")

// Last turn's assistant response
ctx.select("( d1, .block ) +assistant")
```

### Context Position
```javascript
// Pre-context in active head
ctx.select("^ah :pre .block")

// Core container in active head
ctx.select("^ah .cont")

// Core content in active head
ctx.select("^ah .cont > .block")

// Post-context in active head
ctx.select("^ah :post .block")

// All pre-context across conversation
ctx.select("^seq :pre .block")
```

### Sibling Position
```javascript
// First content block in each turn
ctx.select("^seq .seg .block:first")

// Last content block in each turn
ctx.select("^seq .seg .block:last")

// Second content block in each turn
ctx.select("^seq .seg .block:nth(2)")
```

---

## Time-Based Queries

### Snapshot Traversal
```javascript
// Current snapshot (default)
ctx.select("( d1 )")
ctx.select("@t0 ( d1 )")

// Previous snapshot
ctx.select("@t-1 ( d1 )")

// Specific cycle
ctx.select("@c42 ^seq .seg")

// Search across all available snapshots
ctx.select("@* #node123abc")
```

### Historical Analysis
```javascript
// Compare current vs previous assistant response
current = ctx.select("@t0 ( d1, .block ) +assistant")
previous = ctx.select("@t-1 ( d1, .block ) +assistant")

// Track node evolution across time
ctx.select("@* ^seq .block[id='summary_v1']")
// Find when a node ID first appears across snapshots
ctx.select("@* #node123abc")
```

---

## Attribute Filtering

### Lifecycle Management
```javascript
// Nodes about to expire
ctx.select("^root .block[ttl<=1]")

// High priority content
ctx.select("^root .block[priority>=8]")

// Recently created (this cycle)
const c = ctx.cycle
ctx.select(`^root .block[cycle=${c}]`)

// Persistent content
ctx.select("^root .block[ttl=null]")
```

### Content Filtering
```javascript
// Large content blocks (assuming token count attribute)
ctx.select("^root .block[token_count>500]")

// Specific content types
ctx.select("^root .block[kind='image']")
ctx.select("^root .block[kind='tool_result']")
ctx.select("^root .block[kind='summary']")

// Error states
ctx.select("^root .block[status='error']")
ctx.select("^root .block[retry_count>0]")
```

---

## Complex Queries

### Multi-condition Filters
```javascript
// High-priority user messages from recent turns
ctx.select("( d1..d3, .block ) +user[priority>=5]")

// Assistant responses that are about to expire
ctx.select("^seq .block +assistant[ttl<=2]")

// Tool calls with specific status
ctx.select("^root .block[kind='call'] +tool[status='pending']")

// Long-lived system context
ctx.select("^sys .block[ttl=null][priority>=7]")

// Recently created by timestamp (ISO string lexicographic comparison)
ctx.select("^root .block[created_at_iso>='2025-08-01T00:00:00Z']")
```

### Combinatorial Patterns
```javascript
// Direct children only
ctx.select("^seq > .seg")

// All descendants
ctx.select("^seq .block")

// Mixed patterns
ctx.select("^seq .seg > .cont > .block +user")
ctx.select("^ah :pre > .block[kind='context']")
```

---

## Real-World Use Cases

### RAG Integration
```javascript
// Find all context summaries
ctx.select("^root .block[kind='summary']")

// Recent user questions for retrieval
ctx.select("( d1..d3, .block[kind='text'] ) +user")

// Retrieved documents in current turn
ctx.select("^ah .block[kind='retrieved_doc']")

// Expired RAG context that needs refresh
ctx.select("^root .block[kind='rag_context'][ttl<=1]")
```

### Conversation Analysis
```javascript
// All turns in conversation
const turns = ctx.select("^seq .seg")

// User-assistant pairs (assuming alternating pattern)
const userMsgs = ctx.select("^seq .seg .block +user")
const assistantMsgs = ctx.select("^seq .seg .block +assistant")

// Topic changes (assuming topic attribute)
ctx.select("^seq .block[topic!='previous_topic']")

// Long conversations
ctx.select("( d10..d15 )")
```

### Error Handling & Debugging
```javascript
// Failed tool calls
ctx.select("^root .block +tool[status='error']")

// Retried operations
ctx.select("^root .block[retry_count>0]")

// Nodes with validation errors
ctx.select("^root .block[validation_status='failed']")

// Context that exceeded token limits
ctx.select("^root .block[token_limit_exceeded=true]")
```

### Context Management
```javascript
// Budget pressure indicators
const expiring = ctx.select("^root .block[ttl<=3]")
const lowPriority = ctx.select("^root .block[priority<=2]")

// Memory management
const oldContent = ctx.select("( d20..d25 )")
const systemCore = ctx.select("^sys .block[priority>=9]")

// Performance monitoring
const largeNodes = ctx.select("^root .block[size_bytes>10000]")
const slowNodes = ctx.select("^root .block[processing_time>1000]")
```

---

## Advanced Patterns

### Conditional Selection
```javascript
// Select based on conversation state
function getRelevantContext(conversationPhase) {
  switch(conversationPhase) {
    case 'initial':
      return ctx.select("^sys .block[kind='system_prompt']")
    case 'ongoing':
      return ctx.select("( d1..d3, .block )")
    case 'summary':
      return ctx.select("^root .block[kind='summary']")
  }
}
```

### Dynamic Depth Selection
```javascript
// Select varying depths based on context length
function getContextByBudget(maxTokens) {
  for (let depth = 1; depth <= 10; depth++) {
    const selector = `( d1..d${depth} )`
    const nodes = ctx.select(selector)
    const tokens = estimateTokens(nodes)
    if (tokens > maxTokens) {
      return depth - 1
    }
  }
}
```

### Batch Operations
```javascript
// Process all content by type
const textBlocks = ctx.select("^root .block[kind='text']")
const imageBlocks = ctx.select("^root .block[kind='image']")
const toolBlocks = ctx.select("^root .block[kind='tool_call']")

// Update priorities in batch
const criticalNodes = ctx.select("^root .block[urgency='critical']")
// ... apply priority updates
```

---

## Performance Considerations

### Efficient Queries
```javascript
// ✅ Good: Specific region and constraints
ctx.select("( d1..d3, .block ) +user")

// ❌ Avoid: Overly broad queries
ctx.select("@* ^root *")

// ✅ Good: Use most specific selectors
ctx.select("^ah .cont > .block +assistant")

// ❌ Avoid: Multiple descendant traversals
ctx.select("^root .seg .cont .block .nested")
```

### Caching Strategies
```javascript
// Cache frequently accessed nodes
const systemContext = ctx.select("^sys .block[priority>=8]")
const recentTurns = ctx.select("( d1..d3 )")

// Use node IDs for quick lookups
const importantNodeIds = ctx.select("^root .block[priority>=9]").map(n => n.id)
const quickLookup = importantNodeIds.map(id => ctx.select(`#${id}`))
```

---

## Migration Examples

### Converting from Flat Logs
```javascript
// Traditional message array to PACT
const messages = [
  {tag: 'user', content: 'Hello'},
  {tag: 'assistant', content: 'Hi there!'},
  {tag: 'user', content: 'How are you?'}
]

// PACT equivalent queries:
const userMessages = ctx.select("^seq .block +user")
const assistantMessages = ctx.select("^seq .block +assistant")
const allMessages = ctx.select("^seq .block")
```

### From Other Context Systems
```javascript
// Anthropic-style context
const systemPrompt = ctx.select("^sys .block +system")
const conversation = ctx.select("^seq .block +user, ^seq .block +assistant")

// OpenAI-style messages
const chatHistory = ctx.select("^seq .block").map(node => ({
  tag: node.tag,
  content: node.content
}))
```