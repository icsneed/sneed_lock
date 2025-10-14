# ICRC-7 NFT Integration Analysis for SneedLock

## Executive Summary

Converting SneedLock's token locks and position locks to ICRC-7 NFTs is **feasible without major data structure changes**. The main challenge is that ICRC-7 requires immutable token histories (NFTs shouldn't disappear from existence), while the current system deletes expired locks for cleanup. This analysis covers the implementation approach and recommends moving expired locks to a separate archive structure rather than fully eliminating deletion.

## Current System Architecture

### Lock Types

**Token Locks (`Lock`):**
- `lock_id: LockId (Nat)` - Unique identifier
- `amount: Balance` - Amount of tokens locked
- `expiry: Expiry (Nat64)` - Timestamp when lock expires

**Position Locks (`PositionLock`):**
- `lock_id: LockId (Nat)` - Unique identifier
- `position_id: PositionId` - The position being locked
- `expiry: Expiry` - Timestamp when lock expires
- `dex: Dex` - DEX identifier
- `token0, token1: TokenType` - Token pair in the position

### Storage Structure

```
Principal → TokenType → List<Lock>           (Token Locks)
Principal → SwapCanisterId → List<PositionLock>  (Position Locks)
Principal → SwapCanisterId → List<PositionId>    (Position Ownerships)
```

### Current Lock Lifecycle

1. **Creation**: `create_lock()` or `create_or_update_position_lock()`
   - Assigns unique `lock_id` from `stable var current_lock_id: Nat`
   - Stores lock in appropriate data structure
   
2. **Transfer**: `transfer_token_lock_ownership()` or `transfer_position_ownership()`
   - Moves lock from one principal to another
   - Validates lock hasn't expired
   - Also transfers underlying tokens/positions
   
3. **Expiration & Deletion**:
   - `clear_expired_locks()` - Manual cleanup by user
   - `clear_expired_position_locks()` - Manual cleanup by user
   - Called before creating new locks
   - **Permanently deletes expired locks from data structures**

### Key Methods

**Token Lock Methods:**
- `create_lock()` - Create a new token lock
- `transfer_token_lock_ownership()` - Transfer lock to another principal
- `get_token_locks()` - Query locks for caller
- `get_all_token_locks()` - Query all locks (admin)
- `clear_expired_locks()` - Delete expired locks

**Position Lock Methods:**
- `create_or_update_position_lock()` - Create or extend position lock
- `transfer_position_ownership()` - Transfer position and its lock
- `get_position_locks()` - Query position locks for caller
- `get_all_position_locks()` - Query all position locks (admin)
- `clear_expired_position_locks()` - Delete expired position locks

## ICRC-7 Integration Design

### Token ID Encoding Scheme

Since ICRC-7 supports **one collection per canister**, we must encode both lock types in a single token ID space:

```
Token ID Structure:
- Bit 63: Lock type flag (0 = Token Lock, 1 = Position Lock)
- Bits 0-62: lock_id (original lock identifier)

Token ID = (lock_type_bit << 63) | lock_id

Examples:
- Token Lock with lock_id=123:     0x000000000000007B (123)
- Position Lock with lock_id=123:  0x800000000000007B (9223372036854775931)
```

This gives us:
- ~9.2 quintillion token lock IDs (0 to 2^63-1)
- ~9.2 quintillion position lock IDs (2^63 to 2^64-1)

### ICRC-7 Method Mappings

#### Collection Metadata Methods

```motoko
icrc7_collection_metadata() -> vec record { text; Value }
  - name: "SneedLock Lock Collection"
  - symbol: "SNEED-LOCK"
  - description: "Token and Position Locks on SneedLock"
  - total_supply: count of all active locks
  - supply_cap: null (unbounded)
  - max_query_batch_size: 100
  - max_update_batch_size: 10
  - atomic_batch_transfers: false

icrc7_symbol() -> "SNEED-LOCK"
icrc7_name() -> "SneedLock Lock Collection"
icrc7_total_supply() -> count(token_locks) + count(position_locks)
```

#### Token Query Methods

```motoko
icrc7_owner_of(token_ids: vec nat) -> vec opt Account
  For each token_id:
    1. Decode: is_position_lock = (token_id >= 2^63)
    2. Extract: lock_id = token_id & 0x7FFFFFFFFFFFFFFF
    3. If is_position_lock:
         - Iterate through principal_position_locks
         - Find lock with matching lock_id
         - Return owner's Account
       Else:
         - Iterate through principal_token_locks
         - Find lock with matching lock_id
         - Return owner's Account
    4. Return null if not found

icrc7_balance_of(accounts: vec Account) -> vec nat
  For each account:
    - Count locks in principal_token_locks[account.owner]
    - Count locks in principal_position_locks[account.owner]
    - Return sum
  Note: Subaccounts are not used in current system, only principals

icrc7_tokens(prev: opt nat, take: opt nat) -> vec nat
  1. Collect all token_ids from both lock types
  2. Sort by token_id
  3. Implement pagination with prev/take
  4. Return sorted, paginated list

icrc7_tokens_of(account: Account, prev: opt nat, take: opt nat) -> vec nat
  1. Get all locks for account.owner
  2. Encode each lock_id as token_id
  3. Sort and paginate
  4. Return result
```

#### Token Metadata

```motoko
icrc7_token_metadata(token_ids: vec nat) -> vec opt vec record { text; Value }
  For each token_id:
    1. Decode lock type and lock_id
    2. Find the lock
    3. Return metadata:
       - "lock_type": "token" | "position"
       - "lock_id": nat
       - "expiry": nat64
       - "is_expired": bool
       
       Token Lock specific:
       - "token_type": principal (ledger canister)
       - "amount": nat
       
       Position Lock specific:
       - "position_id": nat
       - "swap_canister_id": principal
       - "dex": nat
       - "token0": principal
       - "token1": principal
```

#### Transfer Method

```motoko
icrc7_transfer(args: vec TransferArg) -> vec opt TransferResult
  For each transfer arg:
    1. Decode token_id to get lock type and lock_id
    2. Verify caller owns the lock
    3. Verify lock hasn't expired (return Unauthorized if expired)
    4. If is_position_lock:
         - Call existing transfer_position_ownership logic
         - Return transaction index on success
       Else:
         - Call existing transfer_token_lock_ownership logic
         - Return transaction index on success
    5. Handle errors appropriately
    
  Note: May not be atomic (depends on icrc7_atomic_batch_transfers setting)
```

### Implementation Strategy

**Phase 1: No Data Structure Changes**
- Add ICRC-7 methods as a new layer on top of existing data structures
- All ICRC-7 methods delegate to existing internal functions
- Token ID encoding/decoding happens only at the ICRC-7 API boundary

**Phase 2: Archive Integration (Future)**
- Add expired lock archive structures
- Migrate cleanup logic to move locks to archive instead of deleting
- ICRC-7 methods can still find archived locks by ID

## Impact Analysis: Stopping Lock Deletion

### Current Deletion Behavior

Locks are deleted in three scenarios:
1. **Manual cleanup**: User calls `clear_expired_locks()` or `clear_expired_position_locks()`
2. **Pre-creation cleanup**: Before creating new locks, expired locks are auto-cleared
3. **Transfer failure recovery**: Lock deletion is reverted if transfer fails (safe)

### Consequences of Not Deleting Expired Locks

#### ✅ Benefits
1. **ICRC-7 Compliance**: NFTs remain permanently accessible by ID
2. **Historical Record**: Complete audit trail of all locks ever created
3. **Simplified Logic**: No need to call cleanup functions
4. **Query Integrity**: Token IDs never become invalid

#### ⚠️ Concerns

1. **Unbounded Growth**
   - Every lock created adds to storage forever
   - Worst case: If 1,000 locks/day created, that's 365K locks/year
   - Each Lock: ~50 bytes, each PositionLock: ~80 bytes
   - **Estimated growth**: 18-30 MB/year (manageable)
   
2. **Query Performance**
   - Methods like `get_token_locks()` iterate all locks, including expired ones
   - Current implementation uses `List` (linked list) - O(n) iteration
   - As locks accumulate, queries get slower
   
3. **Iteration Overhead**
   - Many functions iterate locks to find active ones
   - Filtering expired locks on every operation adds cost
   - Especially affects batch operations

#### 💰 Fee Strategy Mitigation
Once lock fees are implemented, the growth rate will naturally decrease:
- Users will only create locks when necessary
- Fee acts as spam prevention
- Growth becomes economically bounded

### Recommended Approach: Expired Lock Archive

Instead of never deleting, **move expired locks to a separate archive structure**:

```motoko
// New archive structures
type ArchivedLock = {
  lock: Lock;
  token_type: TokenType;
  archived_at: Nat64;
};

type ArchivedPositionLock = {
  position_lock: PositionLock;
  swap_canister_id: SwapCanisterId;
  archived_at: Nat64;
};

// Archive storage
stable var archived_token_locks: HashMap<LockId, (Principal, TokenType, ArchivedLock)> = ...;
stable var archived_position_locks: HashMap<LockId, (Principal, SwapCanisterId, ArchivedPositionLock)> = ...;
```

#### Archive Strategy

**When to Archive:**
- Manual user call: `archive_expired_locks()`
- Optional: Automatic archival after N days post-expiry
- Optional: Periodic archival job (e.g., weekly)

**Archive Benefits:**
1. **Fast Active Queries**: Active lock queries only iterate non-expired locks
2. **ICRC-7 Compliance**: Archived locks still accessible by lock_id/token_id
3. **Bounded Active Set**: Active data structures stay lean
4. **Auditability**: Full history preserved with clear separation
5. **Future-Proof**: Can add ICRC-3 block log later for complete history

**Implementation:**
```motoko
// Active lock queries - fast, only iterate active lists
public query func get_token_locks() -> [(LockId, TokenType, Balance, Expiry)] {
  // Only iterates non-archived locks
  // Much faster as expired locks are removed from active lists
}

// ICRC-7 ID lookup - checks both active and archive
public query func icrc7_owner_of(token_ids: vec nat) -> vec opt Account {
  for token_id in token_ids {
    let lock_id = decode_lock_id(token_id);
    // Check active locks first
    let owner = find_in_active_locks(lock_id);
    if (owner == null) {
      // Check archive
      owner := find_in_archive(lock_id);
    };
    return owner;
  }
}

// Archive migration function
public shared func archive_expired_locks() {
  for principal in principals {
    for token_type in token_types {
      let locks = get_locks(principal, token_type);
      let (active, expired) = List.partition(locks, is_expired);
      
      // Move expired to archive
      for lock in expired {
        archived_token_locks.put(lock.lock_id, (principal, token_type, {
          lock = lock;
          token_type = token_type;
          archived_at = now();
        }));
      };
      
      // Keep only active
      update_active_locks(principal, token_type, active);
    }
  }
}
```

### Performance Comparison

| Scenario | Current (Delete) | Never Delete | Archive Approach |
|----------|-----------------|--------------|------------------|
| Create lock | O(1) | O(1) | O(1) |
| Query active locks | O(n) active | O(n) total | O(n) active |
| Query by ID (ICRC-7) | N/A (no ICRC-7) | O(n) total | O(1) hash lookup |
| Transfer lock | O(n) to find | O(n) to find | O(n) to find |
| Storage growth | Bounded | Unbounded | Unbounded (but archived) |
| Historical queries | Not possible | Slow | Fast with archive index |

### Storage Estimates

**Lock Sizes:**
- Token Lock: ~50 bytes (lock_id, amount, expiry)
- Position Lock: ~80 bytes (lock_id, position_id, expiry, dex, 2 principals)
- Archive overhead: +16 bytes (archived_at timestamp)
- HashMap overhead: ~50 bytes/entry

**Growth Scenarios:**

| Locks/Year | Storage/Year (Archive) | 5-Year Storage |
|------------|------------------------|----------------|
| 1,000 | 100 KB | 500 KB |
| 10,000 | 1 MB | 5 MB |
| 100,000 | 10 MB | 50 MB |
| 1,000,000 | 100 MB | 500 MB |

**Assessment**: Even at 100K locks/year, growth is very manageable. At 1M locks/year, might need to consider:
- ICRC-3 block log with external archive canisters
- Prune very old archives (e.g., >5 years) after storing in block log

## Data Structure Changes Required

### Minimal Changes (Phase 1)

```motoko
// Add to Types.mo
public type ICRC7TokenId = Nat;

// Helper functions in main.mo
private func encode_token_id(is_position_lock: Bool, lock_id: LockId): Nat {
  if (is_position_lock) {
    lock_id | 0x8000000000000000  // Set bit 63
  } else {
    lock_id
  }
};

private func decode_token_id(token_id: Nat): (Bool, LockId) {
  let is_position_lock = token_id >= 0x8000000000000000;
  let lock_id = token_id & 0x7FFFFFFFFFFFFFFF;
  (is_position_lock, lock_id)
};
```

### Archive Addition (Phase 2)

```motoko
// Add to Types.mo
public type ArchivedTokenLock = {
  lock: Lock;
  owner: Principal;
  token_type: TokenType;
  archived_at: Nat64;
};

public type ArchivedPositionLock = {
  lock: PositionLock;
  owner: Principal;
  swap_canister_id: SwapCanisterId;
  archived_at: Nat64;
};

// Add to main.mo stable storage
stable var archived_token_locks_stable: [(LockId, ArchivedTokenLock)] = [];
stable var archived_position_locks_stable: [(LockId, ArchivedPositionLock)] = [];

// Add to ephemeral state
transient let archived_token_locks = HashMap.HashMap<LockId, ArchivedTokenLock>(1000, Nat.equal, Hash.hash);
transient let archived_position_locks = HashMap.HashMap<LockId, ArchivedPositionLock>(1000, Nat.equal, Hash.hash);
```

## ICRC-7 Standards Compliance

### Supported Methods (Full Compliance)

✅ `icrc7_collection_metadata()` - Collection info and limits
✅ `icrc7_symbol()` - Collection symbol
✅ `icrc7_name()` - Collection name  
✅ `icrc7_description()` - Collection description
✅ `icrc7_total_supply()` - Total number of locks
✅ `icrc7_owner_of()` - Get lock owner by token ID
✅ `icrc7_balance_of()` - Get lock count for account
✅ `icrc7_tokens()` - Paginated list of all token IDs
✅ `icrc7_tokens_of()` - Paginated list of account's token IDs
✅ `icrc7_token_metadata()` - Get lock metadata
✅ `icrc7_transfer()` - Transfer lock ownership
✅ `icrc10_supported_standards()` - Standards declaration

### Optional Methods (Not Immediately Needed)

⏳ `icrc7_supply_cap()` - No cap, returns null
⏳ `icrc7_logo()` - Can add later
⏳ `icrc7_atomic_batch_transfers()` - Returns false (non-atomic)
⏳ `icrc7_tx_window()` - For deduplication, can add later
⏳ `icrc7_permitted_drift()` - For deduplication, can add later

### Extensions for Future

🔮 **ICRC-37**: Approval and `transfer_from` operations
🔮 **ICRC-3**: Block log for transaction history
🔮 **Custom Extension**: Lock renewal/extension methods

## Implementation Complexity

### Low Complexity ✅
- ICRC-7 metadata methods (symbol, name, supply)
- Token ID encoding/decoding
- Basic query methods (balance_of, tokens_of)

### Medium Complexity ⚠️
- `icrc7_owner_of()` - Need to search across all principals
- `icrc7_tokens()` - Need to collect and sort all token IDs
- `icrc7_transfer()` - Integrate with existing transfer logic

### High Complexity 🔴
- Archive implementation (if chosen)
- ICRC-3 block log integration (future)
- Atomic batch transfers (if implemented)

## Migration Path

### Step 1: Add ICRC-7 Query Methods (Low Risk)
- No state changes
- Add query methods that read existing data
- Test thoroughly

### Step 2: Add ICRC-7 Transfer Method (Medium Risk)
- Wrap existing transfer functions
- Handle token ID encoding/decoding
- Maintain backward compatibility with existing methods

### Step 3: Deprecate Manual Cleanup (Low Risk)
- Stop calling `clear_expired_*` functions
- Locks accumulate but remain accessible
- Monitor storage growth

### Step 4: Implement Archive (High Value, Medium Risk)
- Add archive data structures
- Migrate cleanup to archival
- Update ICRC-7 methods to check archive
- Performance improvements for active queries

### Step 5: ICRC-3 Integration (Future)
- Add block log
- Enable external archive canisters
- Full ICRC standards compliance

## Recommendations

### Immediate Actions

1. **✅ Implement Phase 1**: Add ICRC-7 methods without data structure changes
   - Can be done in one development cycle
   - No breaking changes to existing functionality
   - Provides immediate ICRC-7 compatibility

2. **✅ Stop Deleting Expired Locks**: Remove cleanup calls temporarily
   - Monitor storage growth
   - Assess real-world impact
   - Gather data for archive design

3. **⏳ Add Lock Fees**: Implement economic spam prevention
   - Reduces lock creation rate
   - Makes unbounded growth sustainable
   - Aligns incentives

### Near-Term Actions (3-6 months)

4. **🔄 Implement Archive Strategy**: Move expired locks to archive
   - Keeps active queries fast
   - Maintains ICRC-7 compliance
   - Enables historical analysis
   - Best of both worlds

5. **📊 Add Monitoring**: Track key metrics
   - Total lock count
   - Active vs archived lock ratio
   - Query performance
   - Storage usage

### Future Considerations

6. **🔮 ICRC-3 Block Log**: When archive grows large
   - Enables external archive canisters
   - Full transaction history
   - Industry standard approach

7. **🔮 ICRC-37 Approvals**: If delegation needed
   - Allow users to approve others to manage locks
   - Useful for smart contract integrations

## Conclusion

**Converting SneedLock to ICRC-7 is highly feasible and recommended.**

### Key Findings:

✅ **No Major Data Structure Changes Needed**: ICRC-7 can be implemented as an API layer over existing structures

✅ **Archive Approach is Optimal**: Moving expired locks to archive balances ICRC-7 compliance with performance

✅ **Storage Growth is Manageable**: With lock fees, growth will be economically bounded at sustainable rates

✅ **Phased Implementation**: Can roll out incrementally with low risk

### The Archive Strategy Wins Because:

1. **Performance**: Fast queries for active locks (current use case)
2. **Compliance**: Full ICRC-7 support with permanent token IDs
3. **History**: Complete audit trail preserved
4. **Scalability**: Archive can be indexed efficiently by lock_id
5. **Flexibility**: Easy to add ICRC-3 later for external archival

### Next Steps:

1. Implement ICRC-7 query methods (1-2 weeks)
2. Add ICRC-7 transfer method (1 week)
3. Stop deleting expired locks (1 line change)
4. Implement archive structures (2-3 weeks)
5. Migrate cleanup to archival (1 week)

**Total estimated effort**: 6-8 weeks for full ICRC-7 integration with archive strategy.

