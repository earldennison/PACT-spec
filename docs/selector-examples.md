# PACT Selector Examples

This document provides comprehensive examples of PACT context selectors for real-world use cases.

See [04 – Context Selectors](../spec/04-selectors.md) for the formal specification.

---

## Basic Patterns

### Selecting by Region
```javascript
// All system context
ctx.select("^sys .cb")

// Everything in sealed sequence
ctx.select("^seq")

// Only active head content
ctx.select("^ah .cb")

// All content blocks across entire tree
ctx.select("^root .cb")

// All nodes with created_at_ns beyond a threshold (numeric comparison)
ctx.select("^root .cb[created_at_ns>1724670000000000000]")
```

### Selecting by Type
```javascript
// All message turns
ctx.select("^seq .mt")

// All message containers
ctx.select("^seq .mc")

// All content blocks
ctx.select("^root .cb")

// Namespaced user types and kinds
ctx.select("^seq .cb:summary")            // type token (nodeType="cb", kind="summary")
ctx.select("^root .cb[kind='tool_call']") // kind is an attribute
```

### Selecting by Role
```javascript
// All user messages
ctx.select("^root .cb[role='user']")

// All assistant responses
ctx.select("^root .cb[role='assistant']")

// System messages only
ctx.select("^sys .cb[role='system']")

// Tool calls and results
ctx.select("^root .cb[role='tool']")
```

---

## Positional Selectors

### Recent Turns
```javascript
// Most recent historical turn
ctx.select("^seq .mt:depth(1)")

// Last 3 historical turns
ctx.select("^seq .mt:depth(1,2,3)")

// Last turn's user message
ctx.select("^seq .mt:depth(1) .cb[role='user']")

// Last turn's assistant response
ctx.select("^seq .mt:depth(1) .cb[role='assistant']")
```

### Context Position
```javascript
// Pre-context in active head
ctx.select("^ah :pre .cb")

// Core container in active head
ctx.select("^ah .mc")

// Core content in active head
ctx.select("^ah .mc > .cb")

// Post-context in active head
ctx.select("^ah :post .cb")

// All pre-context across conversation
ctx.select("^seq :pre .cb")
```

### Sibling Position
```javascript
// First content block in each turn
ctx.select("^seq .mt .cb:first")

// Last content block in each turn
ctx.select("^seq .mt .cb:last")

// Second content block in each turn
ctx.select("^seq .mt .cb:nth(2)")
```

---

## Time-Based Queries

### Snapshot Traversal
```javascript
// Current snapshot (default)
ctx.select("^seq .mt:depth(1)")
ctx.select("@t0 ^seq .mt:depth(1)")

// Previous snapshot
ctx.select("@t-1 ^seq .mt:depth(1)")

// Specific cycle
ctx.select("@c42 ^seq .mt")

// Search across all available snapshots
ctx.select("@* #node123abc")
```

### Historical Analysis
```javascript
// Compare current vs previous assistant response
current = ctx.select("@t0 ^seq .mt:depth(1) .cb[role='assistant']")
previous = ctx.select("@t-1 ^seq .mt:depth(1) .cb[role='assistant']")

// Track node evolution across time
ctx.select("@* ^seq .cb[id='summary_v1']")
// Find when a node ID first appears across snapshots
ctx.select("@* #node123abc")
```

---

## Attribute Filtering

### Lifecycle Management
```javascript
// Nodes about to expire
ctx.select("^root .cb[ttl<=1]")

// High priority content
ctx.select("^root .cb[priority>=8]")

// Recently created (this cycle)
const c = ctx.cycle
ctx.select(`^root .cb[cycle=${c}]`)

// Persistent content
ctx.select("^root .cb[ttl=null]")
```

### Content Filtering
```javascript
// Large content blocks (assuming token count attribute)
ctx.select("^root .cb[token_count>500]")

// Specific content types
ctx.select("^root .cb[kind='image']")
ctx.select("^root .cb[kind='tool_result']")
ctx.select("^root .cb[kind='summary']")

// Error states
ctx.select("^root .cb[status='error']")
ctx.select("^root .cb[retry_count>0]")
```

---

## Complex Queries

### Multi-condition Filters
```javascript
// High-priority user messages from recent turns
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user'][priority>=5]")

// Assistant responses that are about to expire
ctx.select("^seq .cb[role='assistant'][ttl<=2]")

// Tool calls with specific status
ctx.select("^root .cb[role='tool'][kind='call'][status='pending']")

// Long-lived system context
ctx.select("^sys .cb[ttl=null][priority>=7]")

// Recently created by timestamp (ISO string lexicographic comparison)
ctx.select("^root .cb[created_at_iso>='2025-08-01T00:00:00Z']")
```

### Combinatorial Patterns
```javascript
// Direct children only
ctx.select("^seq > .mt")

// All descendants
ctx.select("^seq .cb")

// Mixed patterns
ctx.select("^seq .mt > .mc > .cb[role='user']")
ctx.select("^ah :pre > .cb[kind='context']")
```

---

## Real-World Use Cases

### RAG Integration
```javascript
// Find all context summaries
ctx.select("^root .cb[kind='summary']")

// Recent user questions for retrieval
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user'][kind='text']")

// Retrieved documents in current turn
ctx.select("^ah .cb[kind='retrieved_doc']")

// Expired RAG context that needs refresh
ctx.select("^root .cb[kind='rag_context'][ttl<=1]")
```

### Conversation Analysis
```javascript
// All turns in conversation
const turns = ctx.select("^seq .mt")

// User-assistant pairs (assuming alternating pattern)
const userMsgs = ctx.select("^seq .mt .cb[role='user']")
const assistantMsgs = ctx.select("^seq .mt .cb[role='assistant']")

// Topic changes (assuming topic attribute)
ctx.select("^seq .cb[topic!='previous_topic']")

// Long conversations
ctx.select("^seq .mt:depth(10,11,12,13,14,15)")
```

### Error Handling & Debugging
```javascript
// Failed tool calls
ctx.select("^root .cb[role='tool'][status='error']")

// Retried operations
ctx.select("^root .cb[retry_count>0]")

// Nodes with validation errors
ctx.select("^root .cb[validation_status='failed']")

// Context that exceeded token limits
ctx.select("^root .cb[token_limit_exceeded=true]")
```

### Context Management
```javascript
// Budget pressure indicators
const expiring = ctx.select("^root .cb[ttl<=3]")
const lowPriority = ctx.select("^root .cb[priority<=2]")

// Memory management
const oldContent = ctx.select("^seq .mt:depth(20,21,22,23,24,25)")
const systemCore = ctx.select("^sys .cb[priority>=9]")

// Performance monitoring
const largeNodes = ctx.select("^root .cb[size_bytes>10000]")
const slowNodes = ctx.select("^root .cb[processing_time>1000]")
```

---

## Advanced Patterns

### Conditional Selection
```javascript
// Select based on conversation state
function getRelevantContext(conversationPhase) {
  switch(conversationPhase) {
    case 'initial':
      return ctx.select("^sys .cb[kind='system_prompt']")
    case 'ongoing':
      return ctx.select("^seq .mt:depth(1,2,3) .cb")
    case 'summary':
      return ctx.select("^root .cb[kind='summary']")
  }
}
```

### Dynamic Depth Selection
```javascript
// Select varying depths based on context length
function getContextByBudget(maxTokens) {
  for (let depth = 1; depth <= 10; depth++) {
    const selector = `^seq .mt:depth(${Array.from({length: depth}, (_, i) => i + 1).join(',')})`
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
const textBlocks = ctx.select("^root .cb[kind='text']")
const imageBlocks = ctx.select("^root .cb[kind='image']")
const toolBlocks = ctx.select("^root .cb[kind='tool_call']")

// Update priorities in batch
const criticalNodes = ctx.select("^root .cb[urgency='critical']")
// ... apply priority updates
```

---

## Performance Considerations

### Efficient Queries
```javascript
// ✅ Good: Specific region and constraints
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user']")

// ❌ Avoid: Overly broad queries
ctx.select("@* ^root *")

// ✅ Good: Use most specific selectors
ctx.select("^ah .mc > .cb[role='assistant']")

// ❌ Avoid: Multiple descendant traversals
ctx.select("^root .mt .mc .cb .nested")
```

### Caching Strategies
```javascript
// Cache frequently accessed nodes
const systemContext = ctx.select("^sys .cb[priority>=8]")
const recentTurns = ctx.select("^seq .mt:depth(1,2,3)")

// Use node IDs for quick lookups
const importantNodeIds = ctx.select("^root .cb[priority>=9]").map(n => n.id)
const quickLookup = importantNodeIds.map(id => ctx.select(`#${id}`))
```

---

## Migration Examples

### Converting from Flat Logs
```javascript
// Traditional message array to PACT
const messages = [
  {role: 'user', content: 'Hello'},
  {role: 'assistant', content: 'Hi there!'},
  {role: 'user', content: 'How are you?'}
]

// PACT equivalent queries:
const userMessages = ctx.select("^seq .cb[role='user']")
const assistantMessages = ctx.select("^seq .cb[role='assistant']")
const allMessages = ctx.select("^seq .cb")
```

### From Other Context Systems
```javascript
// Anthropic-style context
const systemPrompt = ctx.select("^sys .cb[role='system']")
const conversation = ctx.select("^seq .cb[role='user'], ^seq .cb[role='assistant']")

// OpenAI-style messages
const chatHistory = ctx.select("^seq .cb").map(node => ({
  role: node.role,
  content: node.content
}))
```