# Tries

**Computer Science Fundamentals Series**

Prefix trees · Radix trees · Suffix trees · Autocomplete · Patricia tries · IP routing

*Mid-level software engineer track -- 20 slides*

---

## Table of Contents

1. [What Is a Trie?](#slide-02--what-is-a-trie)
2. [Node Structure & Alphabet Size](#slide-03--node-structure--alphabet-size)
3. [Insertion](#slide-04--insertion)
4. [Search](#slide-05--search)
5. [Deletion](#slide-06--deletion)
6. [Prefix Search & Autocomplete](#slide-07--prefix-search--autocomplete)
7. [Time & Space Complexity](#slide-08--time--space-complexity)
8. [Compressed Tries — Patricia / Radix Trees](#slide-09--compressed-tries--patricia--radix-trees)
9. [Ternary Search Tries](#slide-10--ternary-search-tries)
10. [Suffix Tries](#slide-11--suffix-tries)
11. [Suffix Trees](#slide-12--suffix-trees)
12. [Generalised Suffix Trees](#slide-13--generalised-suffix-trees)
13. [Trie vs Hash Table vs BST](#slide-14--trie-vs-hash-table-vs-bst)
14. [Memory Optimisation Techniques](#slide-15--memory-optimisation-techniques)
15. [Bitwise Tries](#slide-16--bitwise-tries)
16. [Burst Tries](#slide-17--burst-tries)
17. [Applications — Autocomplete & Spell Checking](#slide-18--applications--autocomplete--spell-checking)
18. [Applications — IP Routing & T9 Keyboard](#slide-19--applications--ip-routing--t9-keyboard)
19. [Summary & Further Reading](#slide-20--summary--further-reading)

---

## Slide 02 -- What Is a Trie?

A **trie** (from re**trie**val) is a tree-shaped data structure where each node represents a single character of a key. Keys that share a common prefix share the same path from the root.

### Core properties

- The root node is empty (represents the empty string)
- Each edge is labelled with a character from the alphabet
- Every path from root to a flagged node spells out a stored key
- No node stores the key itself -- the key is implicit in the path
- Also called a **prefix tree** or **digital tree**

### Basic structure

```
           (root)
          /   |   \
         c    d    t
         |    |   / \
         a    o  h   o
        / \   |  |   |
       r   t  g  e   p
       |       
       s   (cat)  (dog)  (the)  (top)
      (cars)
```

> Tries were first described by Edward Fredkin in 1960 and René de la Briandais in 1959. The name "trie" is pronounced like "try" to distinguish it from "tree".

---

## Slide 03 -- Node Structure & Alphabet Size

### Node representation

Each trie node contains:

- An array or map of child pointers -- one per possible character
- A boolean flag `is_end_of_word` marking complete keys
- Optionally, a value associated with the key

### Array-based node (fixed alphabet)

```python
class TrieNode:
    def __init__(self):
        self.children = [None] * 26   # a-z lowercase
        self.is_end = False
        self.value = None              # optional payload
```

### Hash-map-based node (variable alphabet)

```python
class TrieNode:
    def __init__(self):
        self.children = {}             # char -> TrieNode
        self.is_end = False
```

| Approach | Child lookup | Memory per node | Best for |
|----------|-------------|-----------------|----------|
| **Array** | `O(1)` | `O(ALPHABET_SIZE)` | Small, fixed alphabets (a-z) |
| **Hash map** | `O(1)` amortised | `O(actual children)` | Unicode, sparse alphabets |
| **Sorted list** | `O(log k)` | `O(actual children)` | Memory-critical, few children |

> The alphabet size `Sigma` is the dominant factor in trie memory consumption. A 26-slot array wastes space in sparse nodes; a hash map wastes none but adds overhead per entry.

---

## Slide 04 -- Insertion

### Algorithm

1. Start at the root node
2. For each character `c` in the key:
   - If `children[c]` exists, move to that child
   - Otherwise, create a new node, set `children[c]`, and move to it
3. Mark the final node as `is_end_of_word = True`

### Implementation

```python
def insert(root, word):
    node = root
    for char in word:
        idx = ord(char) - ord('a')
        if node.children[idx] is None:
            node.children[idx] = TrieNode()
        node = node.children[idx]
    node.is_end = True
```

### Inserting "cat", "car", "card"

```
After "cat":     After "car":     After "card":
    (root)           (root)           (root)
      |                |                |
      c                c                c
      |                |                |
      a                a                a
      |               / \              / \
      t*             t*   r*          t*   r*
                                           |
                                           d*
```

> Insertion is `O(m)` where `m` is the length of the key. No rebalancing, no hashing, no comparisons beyond single characters.

---

## Slide 05 -- Search

### Exact search

1. Start at the root
2. For each character `c` in the query:
   - If `children[c]` is `None`, return `False`
   - Otherwise, move to that child
3. Return `True` only if the final node has `is_end_of_word = True`

```python
def search(root, word):
    node = root
    for char in word:
        idx = ord(char) - ord('a')
        if node.children[idx] is None:
            return False
        node = node.children[idx]
    return node.is_end
```

### Prefix check

```python
def starts_with(root, prefix):
    node = root
    for char in prefix:
        idx = ord(char) - ord('a')
        if node.children[idx] is None:
            return False
        node = node.children[idx]
    return True   # node exists -- prefix is in the trie
```

> `search("car")` vs `starts_with("car")` -- search requires `is_end = True`; starts_with only requires the path to exist. This distinction is the basis for autocomplete.

---

## Slide 06 -- Deletion

### Algorithm

1. Recursively traverse to the end of the key
2. Unmark `is_end_of_word`
3. On the way back up, delete nodes that:
   - Are no longer end-of-word markers, AND
   - Have no remaining children

### Implementation

```python
def delete(root, word, depth=0):
    if root is None:
        return False
    if depth == len(word):
        if not root.is_end:
            return False
        root.is_end = False
        return not any(root.children)
    idx = ord(word[depth]) - ord('a')
    should_delete = delete(root.children[idx], word, depth + 1)
    if should_delete:
        root.children[idx] = None
        return not root.is_end and not any(root.children)
    return False
```

### Delete "cars" from {car, cars, cat}

```
Before:          After:
  c                c
  |                |
  a                a
 / \              / \
t*   r*          t*   r*
     |
     s*  <-- remove
```

> Deletion is `O(m)`. The recursive cleanup ensures no orphaned chains waste memory. Nodes shared with other keys are preserved.

---

## Slide 07 -- Prefix Search & Autocomplete

### Collecting all words with a given prefix

1. Navigate to the node representing the prefix
2. Run DFS/BFS from that node, collecting all paths to `is_end` nodes

```python
def autocomplete(root, prefix):
    node = root
    for char in prefix:
        idx = ord(char) - ord('a')
        if node.children[idx] is None:
            return []
        node = node.children[idx]
    results = []
    _dfs(node, prefix, results)
    return results

def _dfs(node, current, results):
    if node.is_end:
        results.append(current)
    for i in range(26):
        if node.children[i]:
            _dfs(node.children[i], current + chr(i + ord('a')), results)
```

### Example

```
Prefix "ca" -> navigate to node 'a' under 'c'
DFS from that node finds: "car", "card", "cars", "cat"
```

> Autocomplete is `O(p + k)` where `p` is the prefix length and `k` is the total number of characters in all matching results. No other data structure naturally supports this operation as efficiently.

---

## Slide 08 -- Time & Space Complexity

### Time complexity

| Operation | Trie | Hash Table | Balanced BST |
|-----------|------|-----------|-------------|
| **Insert** | `O(m)` | `O(m)` average | `O(m log n)` |
| **Search** | `O(m)` | `O(m)` average | `O(m log n)` |
| **Delete** | `O(m)` | `O(m)` average | `O(m log n)` |
| **Prefix search** | `O(p + k)` | `O(n * m)` | `O(n * m)` |
| **Sorted traversal** | `O(N)` | `O(n log n)` | `O(N)` |

`m` = key length, `n` = number of keys, `p` = prefix length, `k` = output size, `N` = total characters

### Space complexity

- **Worst case:** `O(n * m * |Sigma|)` -- every key branches at every character
- **Best case:** heavily shared prefixes reduce space significantly
- **Array nodes:** `26 * sizeof(pointer)` per node even if mostly `NULL`
- **Hash map nodes:** proportional to actual branching

> Tries trade space for deterministic `O(m)` operations. The guarantee is per-key, independent of the number of stored keys -- unlike hash tables where collisions degrade performance.

---

## Slide 09 -- Compressed Tries -- Patricia / Radix Trees

### The problem with standard tries

Chains of single-child nodes waste memory:

```
Standard trie for {"romane", "romanus", "romulus"}:
  r -> o -> m -> a -> n -> e*
                         -> u -> s*
                 -> u -> l -> u -> s*

Many nodes with only one child.
```

### Radix tree (compressed trie)

Merge chains of single-child nodes into a single edge with a multi-character label:

```
Radix tree:
       "rom"
      /     \
   "an"     "ulus"*
   /   \
  "e"*  "us"*
```

### Patricia trie

A Patricia trie (Practical Algorithm To Retrieve Information Coded In Alphanumeric) is a specific form of radix tree where every internal node has at least two children. Coined by Donald Morrison in 1968.

- **Space:** `O(n)` nodes for `n` keys -- guaranteed
- **Operations:** same `O(m)` time complexity
- **Trade-off:** edge labels require substring storage or offset pairs

> Radix trees are the standard choice for memory-efficient prefix matching. Used in Linux kernel routing tables, Redis cluster key distribution, and HTTP routers.

---

## Slide 10 -- Ternary Search Tries

### Structure

Each node has three children:

- **Left:** characters less than current
- **Middle:** characters equal to current (continue down the key)
- **Right:** characters greater than current

```
Insert "cat", "cup", "bat":

         c
        /|\
       b  a  u
       |  |  |
       a  t* p*
       |
       t*
```

### Properties

| Feature | Standard Trie | Ternary Search Trie |
|---------|--------------|-------------------|
| **Children per node** | `|Sigma|` array | 3 pointers |
| **Memory** | High (sparse arrays) | Low (3 pointers) |
| **Search time** | `O(m)` | `O(m + log n)` |
| **Prefix search** | Natural | Supported |
| **Sorted order** | In-order of children | In-order traversal |

> TSTs combine the time efficiency of tries with the space efficiency of BSTs. They are particularly effective for large alphabets (Unicode) where array-based tries waste enormous memory.

---

## Slide 11 -- Suffix Tries

### Definition

A suffix trie for a string `S` of length `n` is a trie containing all `n` suffixes of `S`.

### Example: suffix trie for "banana$"

Suffixes: `banana$`, `anana$`, `nana$`, `ana$`, `na$`, `a$`, `$`

```
Insert all suffixes into a standard trie:

        (root)
      / | | \ \
     b  a  n  $
     |  |  |
     a  n  a
     |  |  |
     n  a  n
     ...  ...
```

### Properties

- Every substring of `S` is a prefix of some suffix -- so substrings can be found by prefix search
- **Substring search:** `O(m)` -- just search the trie for the pattern
- **Count occurrences:** count leaves below the matching node
- **Space:** `O(n^2)` -- each suffix has `O(n)` characters

> The `$` sentinel character ensures no suffix is a prefix of another, guaranteeing every suffix ends at a leaf. The quadratic space cost motivates suffix trees.

---

## Slide 12 -- Suffix Trees

### From suffix trie to suffix tree

Apply path compression (radix tree) to the suffix trie -- merge single-child chains into edges labelled with substrings.

```
Suffix tree for "banana$":

           (root)
         /   |    \
       $   a       banana$
          / \       na
        $   na     / \
           / \   $   na$
          $  na$
```

### Key properties

- **Space:** `O(n)` nodes and edges for a string of length `n`
- **Construction:** Ukkonen's algorithm builds the suffix tree in `O(n)` time online
- **Substring search:** `O(m)` for pattern of length `m`
- **Longest repeated substring:** deepest internal node
- **Longest common substring:** with generalised suffix trees

### Ukkonen's algorithm (1995)

- Builds suffix tree incrementally, left to right
- Three key rules: extension rules 1 (leaf extension), 2 (new branch), 3 (do nothing)
- Uses suffix links for amortised linear time
- Active point tracks current position: `(active_node, active_edge, active_length)`

> Suffix trees are one of the most versatile string data structures. They solve dozens of string problems in optimal time, including pattern matching, longest repeated substring, and string statistics.

---

## Slide 13 -- Generalised Suffix Trees

### Definition

A generalised suffix tree (GST) stores the suffixes of multiple strings simultaneously. Each leaf is labelled with both a string index and a suffix position.

### Construction

For strings `S1 = "banana"` and `S2 = "bandana"`:

1. Concatenate with unique sentinels: `banana$1bandana$2`
2. Build suffix tree on the concatenation
3. Mark each leaf with its originating string

### Applications

| Problem | Solution via GST |
|---------|-----------------|
| **Longest common substring** | Deepest internal node with leaves from all strings |
| **Common substrings of k out of n strings** | Internal nodes with leaves from >= k distinct strings |
| **All-pairs suffix-prefix overlap** | Used in genome assembly overlap graphs |
| **Multiple pattern search** | Build GST of patterns; stream text through it |

> Generalised suffix trees are foundational in bioinformatics. Genome assemblers (Velvet, SGA) rely on them to find overlapping reads efficiently. Construction remains `O(N)` where `N` is the total length of all strings.

---

## Slide 14 -- Trie vs Hash Table vs BST

### Feature comparison

| Feature | Trie | Hash Table | Balanced BST |
|---------|------|-----------|-------------|
| **Search** | `O(m)` deterministic | `O(m)` avg, `O(nm)` worst | `O(m log n)` |
| **Insert** | `O(m)` deterministic | `O(m)` avg, `O(nm)` worst | `O(m log n)` |
| **Prefix queries** | Native, `O(p + k)` | Not supported | Not supported |
| **Sorted order** | Natural (DFS) | Requires sorting | Natural (in-order) |
| **Worst-case guarantee** | Always `O(m)` | Degrades with collisions | `O(m log n)` |
| **Memory** | High (pointer-heavy) | Moderate (array + chains) | Moderate |
| **Cache locality** | Poor (pointer chasing) | Good (contiguous array) | Moderate |

### When to use each

- **Trie:** prefix queries, autocomplete, longest prefix match, sorted iteration needed
- **Hash table:** exact-match lookups, no prefix queries, memory-sensitive
- **BST:** sorted order needed, range queries, no prefix operations

> Tries win on prefix operations and worst-case guarantees. Hash tables win on raw speed for exact lookups. BSTs win on range queries. Choose based on your access patterns, not theoretical complexity alone.

---

## Slide 15 -- Memory Optimisation Techniques

### Array compression

Replace fixed-size child arrays with:

- **Bitmask + compact array:** a bitmask records which children exist; a packed array holds only the non-null pointers
- Reduces per-node memory from `O(|Sigma|)` to `O(actual children)`

### Alphabet reduction

- Map characters to smaller ranges: case-insensitive `a-z` = 26 slots instead of 256 for ASCII
- Encode Unicode strings as UTF-8 bytes -- alphabet drops to 256

### Path compression (radix tree)

- Eliminates single-child chains -- reduces node count from `O(n * m)` to `O(n)`

### LOUDS / succinct encoding

- Level-Order Unary Degree Sequence: encode trie topology in ~2 bits per node
- Store labels separately in a flat array
- Near-theoretical-minimum space with rank/select for navigation

### Double-array trie

- Two parallel arrays: `base[]` and `check[]`
- Achieves array-trie speed with compressed memory
- Used in Japanese morphological analysers (MeCab, Darts)

> In practice, a radix tree with hash-map nodes covers most use cases. Succinct tries (LOUDS, DFUDS) are reserved for static dictionaries where minimising memory is paramount.

---

## Slide 16 -- Bitwise Tries

### Binary trie

A trie over the alphabet `{0, 1}`. Keys are interpreted as bit strings. Each level inspects one bit.

```
Insert 5 (101), 2 (010), 7 (111):

         (root)
        0/    \1
       (0)    (1)
      0/ \1  0/ \1
     ( ) (0) (1) ( )
      |   |   |
     ...  2  ...
          *   5*  7*
```

### Bitwise trie (crit-bit tree)

Only branch at "critical bits" where keys differ -- skip identical prefix bits.

### Applications

| Application | Why bitwise trie? |
|------------|------------------|
| **IP routing (longest prefix match)** | Match destination IP against routing prefixes bit by bit |
| **XOR-based nearest neighbour** | Kademlia DHT uses XOR distance on bitwise tries |
| **Integer sets** | Predecessor/successor queries in `O(w)` for `w`-bit keys |
| **Van Emde Boas layout** | Bitwise trie over integer universe with `O(log log U)` operations |

> Linux kernel's Longest Prefix Match for IPv4/IPv6 routing uses a multi-bit trie (LC-trie) -- inspecting several bits per level for speed.

---

## Slide 17 -- Burst Tries

### Motivation

Standard tries have poor cache locality -- every character lookup follows a pointer to a different memory location. Burst tries address this by combining tries with compact containers.

### Structure

- Internal nodes are trie nodes (as usual)
- Leaf nodes are **containers** (sorted arrays, small hash maps, or linked lists) that hold a group of key suffixes
- When a container exceeds a threshold, it **bursts** into trie nodes

```
Burst trie:

     (root)
    /      \
   a        b
   |        |
 [pple,     [at, oy,
  rt,        us]*
  xe]*

* = container (sorted list of suffixes)
```

### Performance

| Feature | Standard Trie | Burst Trie |
|---------|--------------|-----------|
| **Cache misses** | One per character | Amortised -- fewer pointer dereferences |
| **Memory** | High | Lower (containers are compact) |
| **Insert** | `O(m)` | `O(m)` amortised |
| **Search** | `O(m)` | `O(m)` amortised |

> Burst tries were introduced by Heinz, Zobel, and Williams (2002). They consistently outperform standard tries and hash tables on string-heavy workloads due to superior cache behaviour.

---

## Slide 18 -- Applications -- Autocomplete & Spell Checking

### Autocomplete

- **Search engines:** type "how to" -> suggest completions ranked by popularity
- **IDEs:** type `str.` -> list methods starting with `str.`
- **Implementation:** trie with frequency weights; return top-k results by DFS with priority queue

```
Weighted trie for autocomplete:

    (root)
      |
      c
      |
      a
     / \
    r    t  (freq: 500)
    |
    (freq: 300)
    |
    s  (freq: 100)
```

### Spell checking

- Build a trie of dictionary words
- For a query word, search within **edit distance** `d`:
  - Traverse the trie, tracking insertions, deletions, substitutions (Levenshtein)
  - Prune branches where accumulated distance exceeds `d`
- Much faster than checking edit distance against every dictionary word

### Word games

- **Boggle / Scrabble solvers:** DFS on the board guided by trie edges -- prune invalid prefixes immediately
- **Word ladders:** trie-based adjacency lookup

> Google's autocomplete processes billions of queries per day using distributed trie-like structures with frequency-weighted ranking and personalisation layers.

---

## Slide 19 -- Applications -- IP Routing & T9 Keyboard

### IP routing -- longest prefix match

Routers store routing rules as IP prefixes with varying lengths. For each incoming packet, the router must find the longest prefix that matches the destination IP.

```
Routing table:
  10.0.0.0/8     -> Gateway A
  10.1.0.0/16    -> Gateway B
  10.1.1.0/24    -> Gateway C

Packet to 10.1.1.42:
  Matches all three -- longest prefix is /24 -> Gateway C
```

- Binary trie on IP bits gives `O(W)` lookup (`W` = 32 for IPv4, 128 for IPv6)
- Multi-bit tries (LC-trie) inspect 4-8 bits per level for speed
- Hardware implementations (TCAM) achieve single-cycle lookup

### T9 predictive text

- Map phone keypad digits to letters: `2 = abc`, `3 = def`, ...
- Build a trie keyed on digit sequences
- Each node stores the list of possible words matching that digit sequence

```
Input: 4-3-5-5-6
Trie path: 4->3->5->5->6
Matches: "hello", "jello"
```

### Other applications

- **Genome indexing:** suffix trees index DNA sequences for alignment (Bowtie, BWA)
- **Network packet classification:** multi-dimensional tries for firewall rules
- **Compiler symbol tables:** fast prefix-based identifier lookup

> Tries are embedded in systems most developers use daily -- from the router forwarding their packets to the keyboard predicting their words.

---

## Slide 20 -- Summary & Further Reading

### Key takeaways

- A trie is a tree where each path from root to a marked node represents a stored key -- prefix sharing is the core advantage
- Insert, search, and delete are all `O(m)` for key length `m` -- independent of the number of stored keys
- Prefix search and autocomplete are native trie operations with no equivalent in hash tables
- Compressed tries (radix/Patricia) eliminate single-child chains, reducing space to `O(n)` nodes
- Suffix trees solve substring problems in linear time via path-compressed suffix tries
- Ternary search tries balance memory and speed for large alphabets
- Bitwise tries power IP routing and integer data structures
- Burst tries improve cache locality by batching suffixes into compact containers
- Choose tries when prefix operations, sorted order, or worst-case guarantees matter; choose hash tables for raw exact-match speed

### Recommended reading

| Source | Description |
|--------|------------|
| **Sedgewick & Wayne** | *Algorithms, 4th Edition* -- chapter on tries and ternary search tries |
| **Gusfield** | *Algorithms on Strings, Trees, and Sequences* -- definitive reference on suffix trees |
| **Ukkonen (1995)** | "On-line construction of suffix trees" -- the landmark linear-time algorithm |
| **Heinz, Zobel & Williams** | "Burst Tries: A Fast, Efficient Data Structure for String Keys" (2002) |
| **Knuth** | *The Art of Computer Programming, Vol. 3* -- digital searching and retrieval |
| **Morrison (1968)** | "PATRICIA -- Practical Algorithm To Retrieve Information Coded In Alphanumeric" |
