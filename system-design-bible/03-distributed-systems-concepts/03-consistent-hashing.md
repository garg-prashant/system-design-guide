# Consistent Hashing

## The Problem with Naive Hashing

When distributing keys across N servers, the naive approach is:

```
server = hash(key) % N
```

**Problem:** When N changes (adding or removing a server), almost all keys remap to a different server:

```
N=3: key "user123" → hash=456 → 456 % 3 = 0 → Server 0
N=4: key "user123" → hash=456 → 456 % 4 = 0 → Server 0 (lucky!)

N=3: key "item456" → hash=789 → 789 % 3 = 0 → Server 0
N=4: key "item456" → hash=789 → 789 % 4 = 1 → Server 1 (REMAPPED!)
```

When N changes from 3 to 4, approximately (N-1)/N = 75% of keys remap. This causes:
- Cache: Massive cache miss storm
- Database sharding: Expensive data migration

---

## Consistent Hashing

Map both servers AND keys onto the same circular hash space (ring) of size 2^32.

```
        0
       /
  ────/──────────── 2^32
  │              │
  │   Hash Ring  │
  │              │
  ─────────────────
       2^31

Keys and servers both hashed to positions on the ring.
Key is served by the first server encountered going clockwise.
```

### Visual Example

```
          0 (=2^32)
          │
     S1 ──┤──────────
          │          \
          │           \
  S4 ────┤  Ring     ├── K1 (between S1 and K1, K1 → S1)
          │           /
          │          /
     S2 ──┤──────────
          │
     S3 ──┤
          │
     K2 ──┤──── goes to S4 (next server clockwise)
```

### Adding a Server

Only the keys between the new server and its predecessor need to migrate.

```
Before: [S1] [S2] [S3]
Add S4 between S1 and S2:
After: [S1] [S4] [S2] [S3]

Keys that remapped: only those between S1 and S4 (now go to S4 instead of S2)
~1/N of keys remapped (not (N-1)/N)
```

### Removing a Server

Keys that were on the failed server move to its successor.

```
Before: [S1] [S2] [S3]
Remove S2:
After: [S1] [S3]

Keys remapped: only those that were on S2 (now go to S3)
~1/N of keys remapped
```

---

## Virtual Nodes (Vnodes)

A single physical server is represented by multiple positions on the ring.

**Problem with basic consistent hashing:**
- Non-uniform distribution: one server might get more of the ring than others
- Different hardware: a more powerful server should handle more load

**Virtual nodes solution:**

```
Without virtual nodes:
  S1: 25% of ring
  S2: 40% of ring  ← uneven!
  S3: 35% of ring

With 3 virtual nodes per server:
  S1-vn1, S1-vn2, S1-vn3 → distributed around ring
  S2-vn1, S2-vn2, S2-vn3
  S3-vn1, S3-vn2, S3-vn3

  Each server gets ~33% of ring (law of large numbers)
```

**Benefits:**
- Better load distribution
- More gradual rebalancing when a server fails (keys spread across remaining servers)
- Different server capacities: assign more virtual nodes to more powerful servers

**Tradeoff:** More virtual nodes = better distribution but more memory for ring metadata.

**Typical values:** 100-200 virtual nodes per physical node.

---

## Implementation

### In Code (Conceptual)

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, replicas=150):
        self.replicas = replicas
        self.ring = {}          # hash_value → node
        self.sorted_keys = []   # sorted list of hash values
        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        for i in range(self.replicas):
            virtual_key = f"{node}:vn{i}"
            h = self._hash(virtual_key)
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def remove_node(self, node):
        for i in range(self.replicas):
            virtual_key = f"{node}:vn{i}"
            h = self._hash(virtual_key)
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        # Find first node clockwise from key's hash
        idx = bisect.bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
```

---

## Real-World Usage

### Cassandra

- Each node owns a token (range on the ring)
- Virtual nodes (vnodes) distribute ranges across nodes
- Replication: data is stored on RF (replication factor) consecutive nodes

```
Ring with RF=3:
  Key K → Node A (primary), Node B (first replica), Node C (second replica)
```

### DynamoDB

- Similar ring-based partitioning
- Preference list: K successors that are distinct physical hosts
- Quorum reads/writes across this preference list

### Redis Cluster

- 16384 hash slots (not a ring, but similar concept)
- `slot = CRC16(key) % 16384`
- Each node handles a range of slots
- On node addition/removal, slots migrate

### Nginx / HAProxy (Load Balancing)

- `ip_hash` or `hash $request_uri` modules use consistent-ish hashing
- Route same client to same backend (useful for session affinity)

### Memcached Clients

- Client-side consistent hashing to pick the right memcached server
- Libraries: `libmemcached`, Ketama algorithm

---

## Rendezvous Hashing (Highest Random Weight)

Alternative to ring-based consistent hashing.

**Algorithm:**
```
For a given key K, compute score(server_i, K) = hash(server_i, K)
Assign K to the server with the highest score.
```

**Properties:**
- Same: Only 1/N keys remapped when a server is added/removed
- Advantage: Simple, no ring data structure; handles heterogeneous weights easily
- Disadvantage: O(N) lookup (must compute score for all servers)
- Good for: Small N (< 100 servers), load balancers with few backends

---

## Jump Consistent Hashing (Google)

Extremely fast, memory-efficient alternative.

```python
def jump_hash(key: int, num_buckets: int) -> int:
    b, j = -1, 0
    while j < num_buckets:
        b = j
        key = (key * 2862933555777941757 + 1) & 0xffffffffffffffff
        j = int((b + 1) * (1 << 31) / ((key >> 33) + 1))
    return b
```

- O(log N) time, O(1) space
- Minimal disruption on resize
- **Limitation:** Cannot remove arbitrary nodes (only remove the highest-numbered node); suited for stable clusters

---

## Comparison

| Algorithm | Lookup | Memory | Balance | Node removal |
|-----------|--------|--------|---------|--------------|
| Consistent Ring | O(log N) | O(N×vnodes) | Good (with vnodes) | Any node |
| Rendezvous | O(N) | O(N) | Perfect | Any node |
| Jump Hash | O(log N) | O(1) | Perfect | Only last node |

---

## Failure Scenarios

**Hot spot:** Multiple virtual nodes of the same server happen to cluster together → that server gets more than its share of the ring. Fix: More virtual nodes, better hash function.

**Cascading failure:** One server dies → its load moves to successor → successor gets overloaded → it dies → continues cascading.
Fix: Load shedding, circuit breakers, gradual rebalancing, backpressure.

**Data migration during scaling:** When adding capacity, keys move but old server still needs to serve requests while migrating. Fix: Two-phase migration (copy data to new server, then redirect traffic).

---

## Interview Discussion Points

- "For our distributed cache, I'd use consistent hashing with 150 virtual nodes per server. This gives good distribution and limits key migration when we add capacity."
- "When a cache server fails, approximately 1/N of our cache keys will need to be reloaded from the database. For N=10 servers, that's 10% of cache keys — we need to ensure the DB can absorb that burst."
- "We should monitor the distribution of keys per server. If one server is significantly hotter, we may need more virtual nodes or to re-examine our hash function."
- "Redis Cluster uses 16,384 slots rather than a full ring. This makes slot migration explicit and manageable — we can move individual slots rather than resharding everything."
