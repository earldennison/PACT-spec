# PACT (Positioned Adaptive Context Tree)

PACT is a deterministic context substrate that replaces ad‑hoc "chat logs" and template hacks with a first‑class, structured, and auditable context tree. Originally developed for Egregore's agent framework (To be Released), PACT provides a universal foundation that any AI system can rely on for predictable, inspectable, and portable context management.

This README gives a practical overview of why PACT exists, how it works, and what it unlocks for AI system developers. PACT is the definitive reference implementation, demonstrating how deterministic context management enables advanced agent behaviors that weren't possible before.


## Why PACT

### The Chat Log Problem
Traditional chat logs are fundamentally limited for sophisticated AI systems:

**Structural Limitations:**
- **Append-Only Constraints**: No way to modify, insert, or reorganize existing content
- **No Temporal Control**: Content persists forever or gets manually truncated
- **Linear Thinking**: Everything must fit into a sequential message pattern
- **Fragile Memory**: Manual sliding windows lead to context loss and non-deterministic behavior

**Scale Breakdown:**
- **Non-Deterministic Assembly**: Same conversation renders differently across runs
- **Vendor Lock-In**: Each provider handles threading differently, breaking portability
- **Debug Nightmare**: No visibility into "what changed" or "why did context shift"
- **Function Call Amnesia**: Tool results disappear into message history with no state persistence

### Beyond Chat: Stateful Function Calls & Context as Virtual DOM
PACT treats context like a **Virtual DOM for AI conversations** - enabling selective updates, efficient diffing, and intelligent lifecycle management rather than crude append-only logs.

**Traditional Function Calls (Stateless):**
```
Agent: "Call weather_api(location='NYC')"
System: "Current temperature is 72°F"
[Result disappears into chat history]
```

**PACT-Enabled Function Calls (Stateful & Bidirectional):**
- **Persistent State**: Function calls create components that live in the context tree
- **Self-Updating**: Components can modify their own content as external state changes
- **Bidirectional Flow**: External systems can push updates back into agent context
- **Reactive Context**: External events can populate context, processed on next agent cycle
- **Temporal Control**: Function results can expire, cycle, or persist based on relevance
- **Spatial Awareness**: Results position themselves optimally in context hierarchy

**Scaffolds: Stateful Functions with Persistence**
PACT enables a new class of components called "scaffolds" - stateful functions that maintain persistence across conversation turns. These aren't just tools, but **living components** that:
- Maintain internal state between function calls
- Self-render their current status into context
- Provide multiple related operations as a cohesive unit
- Adapt their behavior based on conversation flow

This transforms agents from "request-response processors" into systems with **persistent, evolving state** - like giving them an operating system for memory and capability management.

### PACT's Solution: Context as a Virtual DOM
PACT provides **operating system-level services for AI context**, treating conversation state as a programmable, managed resource:

**Virtual DOM for Conversations:**
- **Selective Updates**: Modify specific parts without rebuilding everything
- **Efficient Diffing**: Compare context states like React compares component trees
- **Lifecycle Management**: Components mount, update, and unmount based on relevance

**Memory Management System:**
- **Adaptive Compression**: Context automatically optimizes for relevance
- **Priority-Based Retention**: Important content persists longer automatically  
- **Semantic Grouping**: Related content clusters together spatially
- **Dynamic Relevance**: "Does this reminder really need to stay in context?"

**Multi-Level Composability:**
- **Micro-Management**: Precise coordinate-based operations  
- **Macro-Orchestration**: High-level workflow patterns
- **Hybrid Approaches**: Switch between detail levels seamlessly
- **Format Flexibility**: Break free from rigid message templates

**Reactive Architecture:**
- **Event-Driven Context**: External systems populate context with real-time updates
- **Asynchronous Processing**: Events queue between agent cycles for LLM consideration
- **Agent Autonomy**: LLM decides how to respond to context changes autonomously
- **Real-Time Awareness**: Agents automatically adapt to changing external conditions

**Core Infrastructure:**
- **Deterministic Structure**: Stable provider bytes, identical renders across runs
- **Temporal Lifecycle**: TTL and cadence give explicit retention semantics
- **Spatial Control**: Insert, update, and organize content at precise coordinates
- **Provider Agnostic**: Same tree works identically across all LLM vendors
- **Bidirectional State**: Components can self-update and receive external modifications
- **First-Class Auditability**: Snapshots, diffs, and selectors for complete visibility


## Core Architecture

PACT implements a sophisticated tree structure with predictable behaviors:

### Universal Coordinate System
- **Infinite Addressability**: Every node has precise coordinates like `(depth, position, offset)`
- **Composable Navigation**: Coordinates compose across tree hops for exact targeting
- **Deterministic Placement**: Same coordinates always reference the same logical position

### Node Types and Structure
- **Container Nodes**: Structural nodes with children along offset axes (pre < 0, core = 0, post > 0)
- **Content Nodes**: Leaf nodes containing actual payload data
- **Canonical Regions**: System header (depth -1), active context (depth 0), conversation history (depth 1+)

### Temporal Management System
- **TTL (Time To Live)**: Components can expire after N conversation turns
- **Cadence System**: Components can reappear cyclically (sticky, cyclic behaviors)
- **Four Component Types**: Permanent, Temporary, Sticky, Cyclic lifecycle patterns
- **ODI (Overlap Demotion Invariant)**: Automatic depth shifting maintains spatial relationships as context evolves

### Deterministic Serialization
- **Provider Agnostic**: Identical tree structure produces identical provider messages regardless of vendor
- **Byte Stability**: Same PACT snapshot always serializes to identical bytes
- **Canonical Ordering**: Predictable linearization from tree to message threads


## Provider Integration

PACT linearizes tree structures to provider message threads in canonical order:

### Linearization Rules
- **Regional Order**: System context → conversation history (oldest→newest) → active context
- **Turn Structure**: Within each turn: pre-context → core message → post-context
- **Role Mapping**: Turn roles become provider message roles (user, assistant, system)
- **Tool Integration**: Tool calls and results are content within their parent turns, not separate roles

### Deterministic Output
Given identical PACT snapshots, the serialized provider bytes are identical across runs, regardless of which LLM provider is used. This enables reproducible agent behavior and reliable A/B testing.


## What PACT Unlocks

### Deterministic Agent Orchestration
Tool execution and component updates happen against a stable, predictable structure. No more context drift between runs - same inputs produce identical outputs every time.

### Reactive Context & Event-Driven Architecture
- **External Event Integration**: External systems can push updates directly into context
- **Asynchronous Processing**: Events populate context between agent cycles
- **Agent-Driven Response**: LLM sees new context and decides how to react
- **Real-Time Adaptation**: Agents respond to changing external conditions automatically

### Complete Audit Trail & Time Travel
- **Immutable Snapshots**: Every conversation cycle can be captured and stored
- **Temporal Diffs**: Compare context states across time with precision
- **Perfect Replay**: Reconstruct any conversation state exactly as it was
- **Debugging**: Trace exactly how context evolved turn by turn

### Sophisticated Memory Management
- **Granular TTL Control**: Components expire after specified conversation turns
- **Cyclic Behaviors**: Content can reappear on schedules (reminders, periodic context)
- **Spatial Stability**: ODI ensures relative positions are maintained as context grows
- **Capacity Management**: Automatic cleanup prevents unbounded context growth

### Advanced Query Capabilities
- **CSS-like Selectors**: Query context structure with familiar syntax
- **Temporal Addressing**: Reference historical states (`@t-3`) and ranges (`@t0..@t-5`)
- **Spatial Navigation**: Precise coordinate-based targeting of any tree position
- **Union Queries**: Combine multiple selection criteria for complex filtering

### Universal Provider Compatibility
One PACT tree works identically across all providers:
- OpenAI (GPT-4, GPT-5)
- Anthropic (Claude)
- Local models (Llama, Mistral)
- Any system with message threading

### Dynamic Context Components & Development Environment
Smart components that live within the context tree:
- **Self-Rendering**: Components generate their own display content
- **State Management**: Persistent data with automatic change detection
- **Operation Exposure**: Component methods become agent tools automatically
- **Lifecycle Integration**: Components participate in TTL and retention management

**Development Environment Integration:**
- **Live Debugging**: Debug agents by examining their context components in real-time
- **State Inspection**: See exactly what agents are "thinking" through persistent state
- **Interactive Development**: Modify agent behavior through component operations
- **Embedded Tooling**: Development tools live within the context tree itself


## Key Concepts in Practice

### Coordinate-Based Addressing
Every piece of content has precise coordinates that compose across tree navigation:
- `(0,0,0)` - Active message core content
- `(1,0,0)` - Previous message (automatically shifted via ODI)
- `(-1,0,0)` - System header region
- `(d2,0,3)` - Third post-context item in message at depth 2

### Temporal Component Lifecycle
Components have sophisticated lifetime management:
- **Permanent**: Live forever, shift with ODI
- **Temporary**: Expire after N turns, deleted permanently
- **Sticky**: Expire and reappear each turn at same relative position
- **Cyclic**: Visible for M turns every N cycles

### CSS-like Selector Language
Query context structure with familiar syntax:
- `^ah .content` - All content in active region
- `^seq .tool_result` - Tool results in conversation history
- `@t-3..@t-1 .content` - Content from 3 turns ago to 1 turn ago
- `.component[ttl=3]` - All components with 3-turn TTL

### Immutable Snapshots
Every conversation state can be captured and compared:
- Snapshots are immutable and byte-stable
- Perfect reconstruction of any historical state
- Temporal diffs show exactly what changed between states
- Complete audit trail for debugging and analysis


## Implementation Architecture

### Core Components
- **Tree Structure**: Hierarchical nodes with precise coordinate addressing
- **Lifecycle Management**: TTL and cadence system for temporal component behavior
- **Selector Engine**: CSS-like query language for tree traversal and filtering
- **Snapshot System**: Immutable state capture and temporal comparison
- **Serialization**: Deterministic conversion to provider message formats

### Key Design Principles
- **Determinism First**: Identical inputs produce identical outputs
- **Spatial Stability**: ODI maintains relative positions as context evolves
- **Temporal Awareness**: Components can expire, cycle, or persist based on conversation turns
- **Provider Agnostic**: Universal serialization works across all LLM providers
- **Debuggability**: Complete visibility into context structure and evolution

### Specification Compliance
PACT follows a formal specification with:
- **Canonical Node Types**: Container and content nodes with defined behaviors
- **Structural Invariants**: Rules that ensure tree consistency and validity
- **Serialization Format**: Standard representation for interoperability
- **Temporal Semantics**: Precise definitions for component lifecycle management

## Why PACT Matters

Traditional chat logs are append-only lists that break down as agent complexity increases. PACT transforms context from "message history" into "programmable memory substrate" - fundamentally changing how we think about AI system design.

### The Paradigm Shift
Instead of asking "How do I fit this into a message?" ask:
- **Dynamic Relevance**: "Does this reminder really need to stay in context?"
- **Format Flexibility**: "Do I really need the same format every time?"
- **Intelligent Persistence**: "What should persist and what should fade?"
- **Contextual Awareness**: "Where should this information live in the conversation?"

### What This Enables
- **Reactive Agent Systems**: Agents automatically respond to external events and system changes
- **Event-Driven Workflows**: Context updates trigger intelligent agent responses
- **Real-Time Integration**: External systems become active participants in conversations
- **Reproducible Debugging**: Exact conversation replay for issue investigation
- **Advanced Memory Patterns**: Components that expire, cycle, or adapt over time
- **Precise Context Surgery**: Insert, update, or remove content at exact locations
- **Cross-Provider Portability**: Same agent logic works identically across LLM vendors
- **Audit Compliance**: Complete trails for enterprise and research requirements
- **Operating System for Agents**: Infrastructure-level services for memory and capability management

PACT represents the evolution from simple chatbots to sophisticated AI systems with predictable, manageable, and auditable context handling - like moving from command-line scripts to full operating systems.

