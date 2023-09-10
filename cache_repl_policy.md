# Cache Replacement Policy

# 1. Cache Replacement Policy's Concerns

* **Insertion Policy**: How does the replacement policy initialize the replacement state of a new line when it is inserted into the cache?
* **Promotion Policy**: How does the replacement policy update the replacement state of a line when it hits in the cache?
* **Aging Policy**: How does the replacement policy update the replacement state of a line when a competing line is inserted or promoted?
* **Eviction Policy**: Given a fixed number of choices, which line does the replacement policy evict?

From book 《Cache Replacement Policies》

# 2. Gem5 Code Structure 

The following diagram is a UML diagram related to cache replacement policies in gem5. Among them, the Indexing Policy is a class closely coupled with the Replacement Policy, so let's first take a look at the IndexingPolicy class.

![](attachments/Pasted%20image%2020230910204252.png)

  
The Indexing Policy determines the mapping of addresses to cache sets and serves as the cornerstone of set-associative caching.
The primary member variable of `BaseIndexingPolicy` is:
- `sets`: This is a two-dimensional array representing cache entries. The first dimension corresponds to cache ways, and the second dimension corresponds to cache sets.
```cpp
std::vector<std::vector<ReplaceableEntry*>> sets;
// sets[set][way];
```
Followings are some member functions:
* Getting and setting entries using set index and way
```cpp
ReplaceableEntry*
BaseIndexingPolicy::getEntry(const uint32_t set, const uint32_t way) const
{
    return sets[set][way];
}

void
BaseIndexingPolicy::setEntry(ReplaceableEntry* entry, const uint64_t index)
{
    // Calculate set and way from entry index
    const std::lldiv_t div_result = std::div((long long)index, assoc);
    const uint32_t set = div_result.quot;
    const uint32_t way = div_result.rem;

    // Sanity check
    assert(set < numSets);

    // Assign a free pointer
    sets[set][way] = entry;

    // Inform the entry its position
    entry->setPosition(set, way);
}
```

* select tag bits & set index bits
```cpp
Addr
BaseIndexingPolicy::extractTag(const Addr addr) const
{
    return (addr >> tagShift);
}

uint32_t
SetAssociative::extractSet(const Addr addr) const
{
    return (addr >> setShift) & setMask;
}
```
* `getPossibleEntries`: This function retrieves a list of possible cache entries based on a given address. It first determines the set index bits specified by the Indexing Policy, and then returns the entries within that set.
```cpp
std::vector<ReplaceableEntry*>
SetAssociative::getPossibleEntries(const Addr addr) const
{
    return sets[extractSet(addr)];
}
```
# 3. Gem5 Replacement Policy's  Key Functions

1. **`reset()`:** This function is called when inserting an entry. It is not called again until the entry is invalidated.

2. **`touch()`:** This function is used for aging and promotion. It updates the replacement policy's statistics, marking the touched block as the most recently accessed.

3. **`invalidate()`:** This function is called when an entry is invalidated, possibly due to a coherence request.

4. **`getVictim()`:** This function is used for eviction. It is called when there is a cache miss and all entries in the set are valid. It selects a victim based on the replacement policy.

   - **Usage of `touch()`:**
     - In the `BaseCache::access` function, `tags->accessBlock(pkt, tag_latency)` is used.
     - In `BaseSetAssoc::accessBlock`, `replacementPolicy->touch(blk->replacementData, pkt)` is called.

   - **Usage of `getVictim()`:**
     - In `BaseCache::handleFill`, `allocateBlock` is called.
     - In `BaseCache::allocateBlock`, `tag->findVictim` is used to find the victim block.
     - In `BaseSetAssoc::findVictim`, `getPossibleEntries` is used to retrieve a set of cache blocks. From this set, a victim is selected.

   - **Usage of `invalidate()`:**
     - In `BaseCache::handleEvictions`, `evictBlock` is called.
     - In `evictBlock`, `invalidateBlock` is used.
     - In `invalidateBlock`, `invalidate` is called.

The diagram illustrates how the replacement policy is invoked in the cache access process.
![](attachments/Pasted%20image%2020230910205019.png)

# 4. How to write a replacement policy

1. Add class `MyReplacementPolicy`
2. Design your replacement data information to be carried
3. Design member functions invalidate(), touch(), reset(), getVictim(), instantiateEntry()

```C++
class MyReplacementPolicy: public Base {
private:
    ... some data
protected:
    struct MyReplData: ReplacementData {
        ...data
        MyReplData()...
    };
public:
    MyReplacementPolicy(const MyReplacementPolicyParams &p);
    ~MyReplacementPolicy() = default;

    void invalidate(const std::shared_ptr<ReplacementData>& replacement_data)
                                                                    override;
    void touch(const std::shared_ptr<ReplacementData>& replacement_data) const
                                                                     override;
    void reset(const std::shared_ptr<ReplacementData>& replacement_data) const
                                                                     override;
    ReplaceableEntry* getVictim(const ReplacementCandidates& candidates) const
                                                                     override;
    std::shared_ptr<ReplacementData> instantiateEntry() override;
};
    
          
```

# 5. LRU

## 5.1 LRUReplData

In the `LRUReplData` structure, a `lastTouchTick` field is added to record the last access time. During replacement, this field is used for sorting.

```C++
struct LRUReplData : ReplacementData
{
    /** Tick on which the entry was last touched. */
    Tick lastTouchTick;

    /**
     * Default constructor. Invalidate data.
     */
    LRUReplData() : lastTouchTick(0) {}
};
```

## 5.2 reset

When data is inserted into a cache set, `lastTouchTick` is set to the current tick.

```C++
void
LRU::reset(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    // Set last touch timestamp
    std::static_pointer_cast<LRUReplData>(
        replacement_data)->lastTouchTick = curTick();
}
```

## 5.3 Touch

Similar to `reset`, this function is used to update the `lastTouchTick` when a block is accessed.

```C++
void
LRU::touch(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    // Update last touch timestamp
    std::static_pointer_cast<LRUReplData>(
        replacement_data)->lastTouchTick = curTick();
}
```

## 5.4 getVictim()

In the LRU replacement policy, the block with the smallest `lastTouchTick` is selected as the victim.

```C++
ReplaceableEntry*
LRU::getVictim(const ReplacementCandidates& candidates) const
{
    // There must be at least one replacement candidate
    assert(candidates.size() > 0);

    // Visit all candidates to find victim
    ReplaceableEntry* victim = candidates[0];
    for (const auto& candidate : candidates) {
        // Update victim entry if necessary
        if (std::static_pointer_cast<LRUReplData>(
                    candidate->replacementData)->lastTouchTick <
                std::static_pointer_cast<LRUReplData>(
                    victim->replacementData)->lastTouchTick) {
            victim = candidate;
        }
    }

    return victim;
}
```

## 5.5 invalidate()

trivial

```C++
void
LRU::invalidate(const std::shared_ptr<ReplacementData>& replacement_data)
{
    // Reset last touch timestamp
    std::static_pointer_cast<LRUReplData>(
        replacement_data)->lastTouchTick = Tick(0);
}
```

# 6.TreeLRU

## 6.1 TreePLRUReplData

Firstly, PLRUTree is defined as an array of bits. 
The index of a node defines its position in the tree, and its relation to its parent. 

Each block has a pointer to a PLRUTree and an index indicating its position among the leaves.

```C++
/**
 * Instead of implementing the tree itself with pointers, it is implemented
 * as an array of bits. The index of the node defines its position in the
 * tree, and its parent. Index 0 represents the root, 1 is its left node,
 * and 2 is its right node. Then, in a BFS fashion this is expanded to the
 * following nodes (3 and 4 are respectively 1's left and right nodes, and
 * 5 and 6 are 2's left and right nodes, and so on).
 *
 * i.e., the following trees are equivalent in this representation:
 *    ____1____
 *  __0__   __1__
 * _0_ _1_ _1_ _0_
 * A B C D E F G H
 *
 * 1 0 1 0 1 1 0
 *
 * Notice that the replacement data entries are not represented in the tree
 * to avoid unnecessary storage costs.
 */
typedef std::vector<bool> PLRUTree;
```


```C++
struct TreePLRUReplData : ReplacementData {
    /**
     * Theoretical index of this replacement data in the tree. In practice,
     * the corresponding node does not exist, as the tree stores only the
     * nodes that are not leaves.
     */
    const uint64_t index;

    /**
     * Shared tree pointer. A tree is shared between numLeaves nodes, so
     * that accesses to a replacement data entry updates the PLRU bits of
     * all other replacement data entries in its set.
     */
    std::shared_ptr<PLRUTree> tree;

    /**
     * Default constructor. Invalidate data.
     *
     * @param index Index of the corresponding entry in the tree.
     * @param tree The shared tree pointer.
     */
    TreePLRUReplData(const uint64_t index, std::shared_ptr<PLRUTree> tree);
};
```

## 6.2 reset()

the same as touch()

```cpp
void
TreePLRU::reset(const std::shared_ptr<ReplacementData>& replacement_data)
const
{
    // A reset has the same functionality of a touch
    touch(replacement_data);
}
```
## 6.3 touch()

Starting from the node corresponding to the touched block, this function adjusts the tree upwards.
![](attachments/Pasted%20image%2020230910205700.png)
```C++
void
TreePLRU::touch(const std::shared_ptr<ReplacementData>& replacement_data) const {
    // Cast replacement data
    std::shared_ptr<TreePLRUReplData> treePLRU_replacement_data =
        std::static_pointer_cast<TreePLRUReplData>(replacement_data);
    PLRUTree* tree = treePLRU_replacement_data->tree.get();

    // Index of the tree entry we are currently checking
    // Make this entry the MRU entry
    uint64_t tree_index = treePLRU_replacement_data->index;

    // Parse and update tree to make every bit point away from the new MRU
    do {
        // Store whether we are coming from a left or right node
        const bool right = isRightSubtree(tree_index);

        // Go to the parent tree node
        tree_index = parentIndex(tree_index);

        // Update node to not point to the touched leaf
        tree->at(tree_index) = !right;
    } while (tree_index != 0);
}
```

## 6.4 getVictim

This function selects the victim based on the TreeLRU replacement policy.
![](attachments/Pasted%20image%2020230910205713.png)

```C++
ReplaceableEntry*
TreePLRU::getVictim(const ReplacementCandidates& candidates) const
{
    // There must be at least one replacement candidate
    assert(candidates.size() > 0);

    // Get tree
    const PLRUTree* tree = std::static_pointer_cast<TreePLRUReplData>(
            candidates[0]->replacementData)->tree.get();

    // Index of the tree entry we are currently checking. Start with root.
    uint64_t tree_index = 0;

    // Parse tree
    while (tree_index < tree->size()) {
        // Go to the next tree entry
        if (tree->at(tree_index)) {
            tree_index = rightSubtreeIndex(tree_index);
        } else {
            tree_index = leftSubtreeIndex(tree_index);
        }
    }

    // The tree index is currently at the leaf of the victim displaced by the
    // number of non-leaf nodes
    return candidates[tree_index - (numLeaves - 1)];
}
```

## 6.5 invalidate()
reverse touch()

```cpp
void
TreePLRU::invalidate(const std::shared_ptr<ReplacementData>& replacement_data)
{
    // Cast replacement data
    std::shared_ptr<TreePLRUReplData> treePLRU_replacement_data =
        std::static_pointer_cast<TreePLRUReplData>(replacement_data);
    PLRUTree* tree = treePLRU_replacement_data->tree.get();

    // Index of the tree entry we are currently checking
    // Make this entry the new LRU entry
    uint64_t tree_index = treePLRU_replacement_data->index;

    // Parse and update tree to make it point to the new LRU
    do {
        // Store whether we are coming from a left or right node
        const bool right = isRightSubtree(tree_index);

        // Go to the parent tree node
        tree_index = parentIndex(tree_index);

        // Update parent node to make it point to the node we just came from
        tree->at(tree_index) = right;
    } while (tree_index != 0);
}
```
# 7. BRRIP

## 7.1 BRRIPReplData
In the `BRRIPReplData` structure:
- `rrpv`: This field represents the Re-Reference Interval Prediction Value. Different values correspond to different re-reference intervals.
    - 0: Near-immediate re-reference interval.
    - max_RRPV-1: Long re-reference interval.
    - max_RRPV: Distant re-reference interval.
- `valid`: A boolean indicating whether the entry is valid.

```C++
struct BRRIPReplData : ReplacementData
{
    /**
     * Re-Reference Interval Prediction Value.
     * Some values have specific names (according to the paper):
     * 0 -> near-immediate re-rereference interval
     * max_RRPV-1 -> long re-rereference interval
     * max_RRPV -> distant re-rereference interval
     */
    SatCounter8 rrpv;

    /** Whether the entry is valid. */
    bool valid;

    /**
     * Default constructor. Invalidate data.
     */
    BRRIPReplData(const int num_bits)
        : rrpv(num_bits), valid(false)
    {
    }
};
```

## 7.2 reset()

When a block is inserted into the cache, the `reset` function is called. It sets the `rrpv` value to either `RRPVMax` or `RRPVMax-1` with a certain probability.


```C++
void
BRRIP::reset(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    std::shared_ptr<BRRIPReplData> casted_replacement_data =
        std::static_pointer_cast<BRRIPReplData>(replacement_data);

    // Reset RRPV
    // Replacement data is inserted as "long re-reference" if lower than btp,
    // "distant re-reference" otherwise
    casted_replacement_data->rrpv.saturate();
    if (random_mt.random<unsigned>(1, 100) <= btp) {
        casted_replacement_data->rrpv--;
    }

    // Mark entry as ready to be used
    casted_replacement_data->valid = true;
}
```

## 7.3 touch()

When a block is touched (accessed), the `touch` function is called. It decreases the `rrpv` counter, and the behavior depends on whether it's in Hit Priority (HP) or Frequency Priority (FP) mode.

```C++
void
BRRIP::touch(const std::shared_ptr<ReplacementData>& replacement_data) const
{
    std::shared_ptr<BRRIPReplData> casted_replacement_data =
        std::static_pointer_cast<BRRIPReplData>(replacement_data);

    // Update RRPV if not 0 yet
    // Every hit in HP mode makes the entry the last to be evicted, while
    // in FP mode a hit makes the entry less likely to be evicted
    if (hitPriority) {
        casted_replacement_data->rrpv.reset();
    } else {
        casted_replacement_data->rrpv--;
    }
}
```


## 7.4 getVictim()

First, it checks if there are any invalid entries; if so, it selects one of them as the victim. 
If there are no invalid entries, it chooses the block with the highest `rrpv` value as the victim and adjusts the `rrpv` values of other entries accordingly （+(RRPV_MAX-RRPV_Victim)).

```C++
ReplaceableEntry*
BRRIP::getVictim(const ReplacementCandidates& candidates) const
{
    // There must be at least one replacement candidate
    assert(candidates.size() > 0);

    // Use first candidate as dummy victim
    ReplaceableEntry* victim = candidates[0];

    // Store victim->rrpv in a variable to improve code readability
    int victim_RRPV = std::static_pointer_cast<BRRIPReplData>(
                        victim->replacementData)->rrpv;

    // Visit all candidates to find victim
    for (const auto& candidate : candidates) {
        std::shared_ptr<BRRIPReplData> candidate_repl_data =
            std::static_pointer_cast<BRRIPReplData>(
                candidate->replacementData);

        // Stop searching for victims if an invalid entry is found
        if (!candidate_repl_data->valid) {
            return candidate;
        }

        // Update victim entry if necessary
        int candidate_RRPV = candidate_repl_data->rrpv;
        if (candidate_RRPV > victim_RRPV) {
            victim = candidate;
            victim_RRPV = candidate_RRPV;
        }
    }

    // Get difference of victim's RRPV to the highest possible RRPV in
    // order to update the RRPV of all the other entries accordingly
    int diff = std::static_pointer_cast<BRRIPReplData>(
        victim->replacementData)->rrpv.saturate();

    // No need to update RRPV if there is no difference
    if (diff > 0){
        // Update RRPV of all candidates
        for (const auto& candidate : candidates) {
            std::static_pointer_cast<BRRIPReplData>(
                candidate->replacementData)->rrpv += diff;
        }
    }

    return victim;
}
```

## 7.5 invalidate()

The `invalidate` function sets the `valid` field to `false` when an entry is invalidated.
```C++

void
BRRIP::invalidate(const std::shared_ptr<ReplacementData>& replacement_data)
{
    std::shared_ptr<BRRIPReplData> casted_replacement_data =
        std::static_pointer_cast<BRRIPReplData>(replacement_data);

    // Invalidate entry
    casted_replacement_data->valid = false;
}
```