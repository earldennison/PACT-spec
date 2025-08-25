[← 07-adoption](07-adoption.md) | [↑ Spec Index](../README.md)

# 08 – Reference Implementations

## 1. Purpose
This appendix provides **reference implementations** for critical PACT algorithms to ensure consistent behavior across implementations. These are normative examples that implementations SHOULD follow for interoperability.

---

## 2. Content Hash Calculation

### 2.1 Algorithm
Content hashing MUST be deterministic and collision-resistant. Implementations MUST use the following algorithm for interoperability:

```python
import hashlib
import json

def calculate_content_hash(node):
    """
    Calculate a stable content hash for a PACT node.
    
    Args:
        node: Node object with content and metadata
        
    Returns:
        str: Hexadecimal hash string (SHA-256)
    """
    # Extract only content-relevant fields, excluding metadata
    hashable_content = {
        'content': node.get('content', ''),
        'kind': node.get('kind', ''),
        'role': node.get('role', ''),
        # Include any custom content attributes
        **{k: v for k, v in node.items() 
           if k.startswith('content_') or k.startswith('data_')}
    }
    
    # Canonical JSON serialization (sorted keys, no whitespace)
    canonical_json = json.dumps(hashable_content, 
                               sort_keys=True, 
                               separators=(',', ':'),
                               ensure_ascii=True)
    
    # SHA-256 hash
    return hashlib.sha256(canonical_json.encode('utf-8')).hexdigest()
```

### 2.2 Excluded Fields
Content hash MUST NOT include:
- Lifecycle metadata: `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`
- Tree structure: `id`, `offset`, `parent_id`, `children`
- Implementation-specific: internal pointers, cache values, etc.

### 2.3 Hash Stability
Content hash MUST remain stable across:
- Node movement within tree
- TTL changes
- Priority adjustments
- Timestamp updates

---

## 3. Concurrent Access Patterns

### 4.1 Threading Model
PACT implementations SHOULD assume single-threaded access during commits, but MAY support concurrent readers:

```python
import threading
from typing import Optional

class PACTTree:
    def __init__(self):
        self._snapshot_lock = threading.RLock()
        self._current_snapshot: Optional[Dict] = None
        self._read_count = 0
        
    def get_snapshot(self, address: str = "@t0"):
        """Thread-safe snapshot access for selectors/diffs."""
        with self._snapshot_lock:
            # Multiple concurrent readers allowed
            self._read_count += 1
            try:
                return self._get_snapshot_impl(address)
            finally:
                self._read_count -= 1
    
    def commit_cycle(self, mutations):
        """Exclusive write access during commit."""
        with self._snapshot_lock:
            # Wait for active readers to complete
            while self._read_count > 0:
                time.sleep(0.001)  # Brief yield
            
            # Atomic commit
            new_snapshot = self._apply_mutations(mutations)
            self._current_snapshot = new_snapshot
            return new_snapshot
```

### 4.2 Memory Safety
Node references and TTL management:

```python
from weakref import WeakSet

class NodeReference:
    """Safe reference to PACT nodes with TTL interaction."""
    
    def __init__(self, node_id: str, tree: PACTTree):
        self.node_id = node_id
        self._tree_ref = weakref.ref(tree)
        
        # Register with tree to prevent premature TTL expiry
        tree._register_reference(self)
    
    def get_node(self):
        """Get current node, returns None if expired/pruned."""
        tree = self._tree_ref()
        if tree is None:
            return None
        return tree.get_node_by_id(self.node_id)
    
    def __del__(self):
        """Cleanup reference on destruction."""
        tree = self._tree_ref()
        if tree is not None:
            tree._unregister_reference(self)
```

---

## 5. Canonical Ordering Implementation

### 5.1 Comparison Function
Deterministic sibling ordering implementation:

```python
from functools import cmp_to_key
from typing import List, Dict, Any

def compare_siblings(node_a: Dict[str, Any], node_b: Dict[str, Any]) -> int:
    """
    Compare two sibling nodes according to PACT canonical order.
    
    Returns:
        -1 if node_a < node_b
         0 if node_a == node_b  
         1 if node_a > node_b
    """
    # 1. Compare offset (ascending)
    offset_a = node_a.get('offset', 0)
    offset_b = node_b.get('offset', 0)
    if offset_a != offset_b:
        return -1 if offset_a < offset_b else 1
    
    # 2. Compare created_at_ns (ascending)
    time_a = node_a.get('created_at_ns', 0)
    time_b = node_b.get('created_at_ns', 0)
    if time_a != time_b:
        return -1 if time_a < time_b else 1
    
    # 3. Compare creation_index (ascending)
    index_a = node_a.get('creation_index', 0)
    index_b = node_b.get('creation_index', 0)
    if index_a != index_b:
        return -1 if index_a < index_b else 1
    
    # 4. Compare id (lexicographic ascending)
    id_a = node_a.get('id', '')
    id_b = node_b.get('id', '')
    if id_a < id_b:
        return -1
    elif id_a > id_b:
        return 1
    else:
        return 0

def sort_siblings(siblings: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """Sort siblings according to PACT canonical order."""
    return sorted(siblings, key=cmp_to_key(compare_siblings))
```

### 5.2 Tree Traversal
Canonical depth-first traversal:

```python
def traverse_canonical(node: Dict[str, Any], visit_fn):
    """
    Traverse PACT tree in canonical order.
    
    Args:
        node: Current node
        visit_fn: Function called for each node
    """
    # Visit current node
    visit_fn(node)
    
    # Visit children in canonical order
    children = node.get('children', [])
    for child in sort_siblings(children):
        traverse_canonical(child, visit_fn)
```

---

## 6. Serialization Reference

### 6.1 Provider-Agnostic Serialization
Byte-stable serialization for snapshots:

```python
import json
from typing import Dict, Any

def serialize_snapshot(tree: Dict[str, Any]) -> bytes:
    """
    Serialize PACT tree to byte-stable format.
    
    Args:
        tree: PACT tree root
        
    Returns:
        bytes: Canonical serialization
    """
    def normalize_node(node: Dict[str, Any]) -> Dict[str, Any]:
        """Normalize node for serialization."""
        normalized = {}
        
        # Required fields in fixed order
        for field in ['id', 'nodeType', 'offset', 'ttl', 'priority', 
                     'cycle', 'created_at_ns', 'created_at_iso']:
            if field in node:
                normalized[field] = node[field]
        
        # Content fields
        for field in ['content', 'role', 'kind']:
            if field in node:
                normalized[field] = node[field]
                
        # Children in canonical order
        if 'children' in node:
            normalized['children'] = [
                normalize_node(child) 
                for child in sort_siblings(node['children'])
            ]
        
        return normalized
    
    normalized_tree = normalize_node(tree)
    canonical_json = json.dumps(normalized_tree, 
                               sort_keys=True,
                               separators=(',', ':'),
                               ensure_ascii=True)
    
    return canonical_json.encode('utf-8')
```

---

## 7. Conformance Testing

### 7.1 Reference Test Cases
Implementations SHOULD pass these reference tests:

```python
def test_content_hash_stability():
    """Test that content hash is stable across metadata changes."""
    node1 = {
        'id': 'test1',
        'content': 'Hello world',
        'role': 'user',
        'ttl': 5,
        'created_at_ns': 1000000
    }
    
    node2 = {
        'id': 'test2',  # Different ID
        'content': 'Hello world',
        'role': 'user', 
        'ttl': 10,      # Different TTL
        'created_at_ns': 2000000  # Different timestamp
    }
    
    # Content hashes should be identical
    assert calculate_content_hash(node1) == calculate_content_hash(node2)

def test_canonical_ordering():
    """Test deterministic sibling ordering."""
    siblings = [
        {'id': 'c', 'offset': 0, 'created_at_ns': 3000},
        {'id': 'a', 'offset': -1, 'created_at_ns': 1000}, 
        {'id': 'b', 'offset': 0, 'created_at_ns': 2000},
        {'id': 'd', 'offset': 1, 'created_at_ns': 4000}
    ]
    
    sorted_siblings = sort_siblings(siblings)
    expected_order = ['a', 'b', 'c', 'd']  # offset, then time, then id
    actual_order = [node['id'] for node in sorted_siblings]
    
    assert actual_order == expected_order

def test_serialization_stability():
    """Test byte-stable serialization."""
    tree = create_test_tree()
    
    # Multiple serializations should be identical
    bytes1 = serialize_snapshot(tree)
    bytes2 = serialize_snapshot(tree)
    
    assert bytes1 == bytes2
```

---

## 8. Implementation Guidelines

### 8.1 Performance Considerations
- **Token counting:** Cache results where content unchanged
- **Content hashing:** Compute lazily, cache aggressively  
- **Canonical ordering:** Pre-sort during tree construction
- **Serialization:** Stream large trees to avoid memory pressure

### 8.2 Error Handling
Implementations MUST handle these error conditions consistently:
- **Invalid content:** MUST return empty string hash, SHOULD log warning
- **Missing metadata:** MUST use defaults (priority=0, ttl=None, created_at_ns=0)
- **Circular references:** MUST detect and reject during tree construction
- **Hash collisions:** MUST log if detected (extremely unlikely with SHA-256)

---

[← 07-adoption](07-adoption.md) | [↑ Spec Index](../README.md)