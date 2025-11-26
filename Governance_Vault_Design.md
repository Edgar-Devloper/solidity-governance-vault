# Time-Locked Governance Vault - Technical Design

**Edgar Velazquez** | edgara.velazquezg@gmail.com  
November 26, 2025

---

## A. Implementation Timeline

Based on my experience with similar vault contracts and upgradeable systems, here's a realistic breakdown:

**Week 1 (5 days):**
- Days 1-2: Core vault logic (lock, unlock, voting power calculation)
- Days 3-4: Unit tests + edge cases (time boundaries, amount limits)
- Day 5: UUPS proxy setup and initialization logic

**Week 2 (3-4 days):**
- Days 1-2: Fuzzing with Foundry (invariant tests are critical here)
- Day 3: Integration testing on Arbitrum fork
- Day 4: Documentation and final security review

**Total: ~2 weeks**

This assumes I'm working with an existing ERC20 governance token and don't need to mess with the frontend. If the existing treasury has specific integration requirements, add 1-2 days for that.

---

## B. Contract Architecture

### Storage Design

I'd go with a simple, gas-efficient approach:

```solidity
pragma solidity ^0.8.20;

contract GovernanceVault {
    struct Lock {
        uint128 amount;        // Enough for most token supplies
        uint64 lockEnd;        // Timestamp when unlock happens
        uint64 lockDuration;   // Original duration for decay calculation
    }
    
    mapping(address => Lock) public locks;
    IERC20 public immutable governanceToken;
    uint256 public totalLocked;
    
    uint256 public constant MIN_LOCK = 30 days;
    uint256 public constant MAX_LOCK = 1460 days; // ~4 years
}
```

**Why this structure?**
- Packing `amount`, `lockEnd`, and `lockDuration` into one storage slot saves gas
- `uint128` handles up to 3.4e38 tokens (plenty for governance tokens)
- `immutable` for the token address saves ~2100 gas per read
- One lock per user keeps voting power queries O(1)

### Voting Power Math

The requirement is that voting power decays as the lock approaches expiry. Here's my approach:

```
At creation: votingPower = amount × multiplier (based on duration)
Over time: votingPower decreases linearly to 1x at expiry
```

Implementation:

```solidity
function votingPower(address account) external view returns (uint256) {
    Lock memory lock = locks[account];
    if (lock.amount == 0) return 0;
    
    // After expiry, just return base amount
    if (block.timestamp >= lock.lockEnd) {
        return uint256(lock.amount);
    }
    
    // Linear decay formula
    uint256 timeLeft = lock.lockEnd - block.timestamp;
    uint256 boost = (timeLeft * 3e18) / lock.lockDuration; // 3x max boost
    
    return (lock.amount * (1e18 + boost)) / 1e18;
}
```

**Design choices:**
- Max 4x multiplier (1x base + 3x boost) for 4-year locks
- Linear decay is simple and predictable for users
- Pure math, no external calls - cheap to query
- The 1e18 scaling prevents precision loss

### Single Lock vs Multiple Locks

I'm going with **one lock per address**. Here's why:

**Pros:**
- Constant-time voting power lookups (critical for governance)
- Way simpler to reason about and audit
- No loops = no gas attacks
- Users understand "one commitment at a time"

**Cons:**
- Users can't have multiple strategies simultaneously
- Have to use multiple wallets for different lock periods

In my experience, the gas savings and security benefits outweigh the flexibility loss. Users who want complex strategies can use multiple addresses or just extend their existing lock.

### Lock Extension Logic

Users should be able to:
1. Add more tokens to existing lock
2. Extend the lock duration

```solidity
function extend(uint256 addAmount, uint256 newDuration) external {
    Lock storage lock = locks[msg.sender];
    require(lock.amount > 0, "No active lock");
    
    if (addAmount > 0) {
        governanceToken.transferFrom(msg.sender, address(this), addAmount);
        lock.amount += uint128(addAmount);
        totalLocked += addAmount;
    }
    
    if (newDuration > 0) {
        require(newDuration >= MIN_LOCK && newDuration <= MAX_LOCK);
        uint256 newEnd = block.timestamp + newDuration;
        require(newEnd > lock.lockEnd, "Must extend, not reduce");
        
        lock.lockEnd = uint64(newEnd);
        lock.lockDuration = uint64(newDuration);
    }
    
    emit LockExtended(msg.sender, lock.amount, lock.lockEnd);
}
```

---

## C. Upgradeability Strategy

### Proxy Pattern: UUPS

I'd use **UUPS** (EIP-1822) instead of the Transparent Proxy pattern.

**Why UUPS for this use case:**
- On L2s, the extra delegatecall overhead in Transparent proxies adds up
- Upgrade logic lives in the implementation, which is cleaner IMO
- Better gas costs for users (they don't pay for proxy routing)
- More control over upgrade authorization

**Trade-off:** You have to be careful not to brick the upgrade function. But with proper testing and maybe a guardian multisig, this is manageable.

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/AccessControlUpgradeable.sol";

contract GovernanceVault is UUPSUpgradeable, AccessControlUpgradeable {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    
    function initialize(address token) external initializer {
        __UUPSUpgradeable_init();
        __AccessControl_init();
        
        governanceToken = IERC20(token);
        _grantRole(ADMIN_ROLE, msg.sender);
    }
    
    function _authorizeUpgrade(address) internal override onlyRole(ADMIN_ROLE) {}
}
```

### Handling Upgrades Without Breaking State

**Storage layout discipline is critical here.**

Rules I follow:
1. Never reorder existing state variables
2. Never change types of existing variables
3. Always append new variables at the end
4. Use storage gaps to reserve space

```solidity
// V1
contract GovernanceVaultV1 {
    mapping(address => Lock) public locks;
    IERC20 public governanceToken;
    uint256 public totalLocked;
    uint256[47] private __gap;  // Reserve slots for future use
}

// V2 - safe upgrade
contract GovernanceVaultV2 {
    mapping(address => Lock) public locks;      // Same position
    IERC20 public governanceToken;              // Same position
    uint256 public totalLocked;                 // Same position
    
    // New features
    mapping(address => address) public delegates;  // New slot
    uint256[46] private __gap;                     // Reduced gap
}
```

I'd also use Hardhat's `@openzeppelin/hardhat-upgrades` plugin to validate storage layout before deploying upgrades.

### Governance Access Control

For production, this should be behind a timelock + multisig:

```solidity
bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
address public timelock;  // Set in initializer

function setTimelock(address newTimelock) external onlyRole(ADMIN_ROLE) {
    timelock = newTimelock;
}
```

Governance flow:
1. Proposal goes through token-weighted voting
2. After passing, queued in timelock (48h delay)
3. Executed by timelock contract
4. Upgrader role checks that caller is timelock

---

## D. Security Analysis

### 1. Reentrancy

**Risk:** If the governance token is malicious or has hooks (like ERC777), it could reenter during transfers.

**Fix:**
- OpenZeppelin's `ReentrancyGuard` on lock/unlock/extend
- CEI pattern (checks-effects-interactions)
- Update state before any external calls

```solidity
function withdraw() external nonReentrant {
    Lock memory lock = locks[msg.sender];
    require(block.timestamp >= lock.lockEnd);
    
    uint256 amt = lock.amount;
    delete locks[msg.sender];  // Update state first
    totalLocked -= amt;
    
    governanceToken.transfer(msg.sender, amt);  // External call last
}
```

### 2. Time Manipulation

**Risk:** Block producers can manipulate `block.timestamp` by ~15 seconds on L2s.

**Mitigation:**
- Our lock periods are in days/months, so 15 seconds is negligible
- Don't use timestamps for anything sub-minute critical
- L2s have more predictable block times anyway

Not a major concern for this use case.

### 3. Arithmetic Issues

**Risk:** Overflow/underflow in voting power calculations, especially with large token amounts.

**Mitigation:**
- Solidity 0.8+ has built-in overflow checks
- Validate casts: `require(amount <= type(uint128).max)`
- Fuzz test with max values

```solidity
require(amount <= type(uint128).max, "Amount overflow");
lock.amount += uint128(amount);
```

### 4. Storage Collisions on Upgrade

**Risk:** This is the big one with proxies. Mess up storage layout in V2, corrupt all user locks.

**Mitigation:**
- Storage gaps (shown above)
- Automated checks with OpenZeppelin upgrades plugin
- Manual review of storage diffs before upgrade
- Test upgrades on fork before mainnet

I've seen this go wrong before. It's unrecoverable if you corrupt user data.

### 5. Precision Loss

**Risk:** Integer division truncates. In voting power calculations, this could underestimate power.

**Mitigation:**
- Always multiply before dividing
- Use 1e18 scaling consistently
- Test with 1 wei amounts

```solidity
// Bad: (amount / duration) * boost
// Good: (amount * boost) / duration
uint256 power = (amount * boost) / 1e18;
```

### 6. Griefing / State Bloat

**Risk:** Attacker creates thousands of tiny locks to bloat state.

**Mitigation:**
- Minimum lock amount (e.g., 100 tokens)
- Single lock per address limits attack surface
- Could add a small creation fee if needed

```solidity
require(amount >= MIN_LOCK_AMOUNT, "Lock too small");
```

### 7. Centralization / Admin Key Compromise

**Risk:** If admin keys get compromised, attacker could upgrade to malicious contract.

**Mitigation:**
- Multisig for admin role (3/5 or similar)
- Timelock delay (48+ hours)
- Emergency pause mechanism
- Monitor upgrade proposals on-chain

This is probably the biggest real-world risk. Social engineering, compromised keys, etc. No smart contract design fully solves this - it's operational security.

---

## E. Testing Approach

### Unit Tests (Hardhat/Foundry)

I'd structure tests by functionality:

**Lock Creation:**
```javascript
- Lock with valid duration creates correctly
- Revert on duration < MIN_LOCK
- Revert on duration > MAX_LOCK  
- Revert on amount too small
- Token transfer happens correctly
- LockCreated event emitted
```

**Voting Power:**
```javascript
- 4-year lock = 4x power at creation
- 1-month lock = ~1.08x power at creation
- Power decays linearly
- Power = 1x after expiry
- Zero power for accounts with no lock
```

**Time-based:**
```javascript
- Can't withdraw before lockEnd
- Can withdraw exactly at lockEnd
- Can withdraw after lockEnd
- Power calculation at 10%, 50%, 90% through lock
```

**Extensions:**
```javascript
- Can add tokens to existing lock
- Can extend duration
- Can't reduce duration
- Voting power updates correctly after extension
```

**Upgrades:**
```javascript
- V2 upgrade preserves all locks
- Storage layout stays compatible
- Only admin can upgrade
- Can't initialize twice
```

### Fuzzing (Foundry)

Invariant tests are crucial for catching edge cases:

```solidity
// Invariant: Sum of all locks equals totalLocked
function invariant_totalLocked() public {
    uint256 sum = 0;
    for (uint i = 0; i < users.length; i++) {
        sum += vault.locks(users[i]).amount;
    }
    assertEq(vault.totalLocked(), sum);
}

// Invariant: Voting power bounded by [1x, 4x]
function invariant_powerBounds() public {
    for (uint i = 0; i < users.length; i++) {
        uint256 locked = vault.locks(users[i]).amount;
        uint256 power = vault.votingPower(users[i]);
        assertGe(power, locked);
        assertLe(power, locked * 4);
    }
}
```

### Edge Cases I'd Specifically Test

1. **Timestamp boundaries:**
   - Lock ending at `type(uint64).max`
   - Withdrawal exactly 1 second after lockEnd

2. **Large values:**
   - Lock `type(uint128).max` tokens
   - 1000 users locking at same block (gas costs)

3. **Griefing:**
   - Front-run extension with tiny lock
   - Try to extend with 0 amount and 0 duration
   - Lock 1 wei for max duration

4. **Upgrade paths:**
   - Upgrade while locks are active
   - Multiple sequential upgrades (V1→V2→V3)
   - Try to reinitialize after upgrade

### Integration Testing

- Deploy on Arbitrum Goerli testnet
- Test with a real ERC20 token
- Integrate with a mock Governor contract
- Measure actual gas costs on L2
- Test upgrade flow end-to-end

---

## Additional Notes

**Gas Optimizations:**
- Storage packing saves ~15k gas per lock
- `immutable` saves ~2100 gas per voting power query
- Single lock model avoids loop costs
- UUPS saves ~1k gas per call vs Transparent

**Alternative Approaches I Considered:**

1. **Multiple locks per user:** More flexible but way more complex. Query costs become O(n) which breaks governance systems that need to check many users.

2. **NFT-based locks:** Could make locks transferable via ERC721. Interesting for secondary markets but adds complexity and wasn't in requirements.

3. **Curve-based decay:** Instead of linear, use a curve (exponential, etc). More sophisticated but harder to reason about for users.

**Future Enhancements (if needed later):**
- Delegation (users can delegate voting power)
- Reward distribution to lockers
- Early exit with penalty
- Lock merging/splitting

---

Let me know if you want me to clarify any design decisions or dive deeper into specific areas.

Edgar Velazquez  
edgara.velazquezg@gmail.com
