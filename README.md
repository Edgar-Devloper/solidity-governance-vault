# Time-Locked Governance Vault - Technical Design

Smart contract architecture for a governance vault with time-based voting power decay.

## Overview

Users lock governance tokens for specified periods (1-48 months) and receive boosted voting power that decays linearly over time. This design was created as part of a technical assessment for a DeFi protocol position.

## Key Features

- **UUPS Upgradeable Proxy Pattern** - Gas-efficient upgrades optimized for L2
- **O(1) Voting Power Queries** - Single lock per user for constant-time lookups
- **Linear Decay Mechanism** - Voting power ranges from 4x (at lock) to 1x (at expiry)
- **L2-Optimized** - Designed for Arbitrum/Optimism deployment
- **Comprehensive Security Analysis** - 7 attack vectors identified with mitigations

## Architecture Highlights

### Storage Design
```solidity
struct Lock {
    uint128 amount;        // Tokens locked
    uint64 lockEnd;        // Expiration timestamp
    uint64 lockDuration;   // Original duration for decay calculation
}
```

### Voting Power Formula
```
votingPower = amount × (1 + (timeRemaining / originalDuration) × 3)

At creation (48 months): 4x power
At halfway point: ~2.5x power
At expiry: 1x power
```

### Upgradeability Strategy

- UUPS pattern for L2 gas efficiency
- Storage gaps for safe upgrades
- Access control with timelock integration
- Tested upgrade paths (V1 → V2 → V3)

## Security Considerations

The design addresses:

1. **Reentrancy Attacks** - ReentrancyGuard + CEI pattern
2. **Time Manipulation** - Day-level granularity mitigates miner manipulation
3. **Arithmetic Issues** - Solidity 0.8+ overflow checks + safe casting
4. **Storage Collisions** - Storage gap pattern for upgrade safety
5. **Precision Loss** - Multiply-before-divide + 1e18 scaling
6. **Griefing/Spam** - Minimum lock amount + single lock per user
7. **Admin Key Compromise** - Multisig + timelock + monitoring

## Testing Strategy

- **Unit Tests** - Lock creation, voting power, withdrawals, extensions
- **Fuzzing** - Foundry invariant tests for totalLocked consistency and power bounds
- **Integration Tests** - L2 fork testing with real ERC20 tokens
- **Edge Cases** - Boundary timestamps, max values, upgrade scenarios

## Tech Stack

- Solidity 0.8.20+
- OpenZeppelin Upgradeable Contracts
- Foundry (testing & fuzzing)
- Hardhat (deployment)

## Documentation

See [Governance_Vault_Design.md](./Governance_Vault_Design.md) for complete technical specification including:

- Implementation timeline (~2 weeks)
- Contract architecture details
- Upgradeability patterns
- Security analysis
- Testing approach

## Design Decisions

**UUPS over Transparent Proxy**
- Lower gas costs for users (critical for L2)
- Upgrade logic in implementation (cleaner architecture)
- Better control over upgrade authorization

**Single Lock Per User**
- Constant-time voting power queries
- Simpler state management
- Reduced attack surface
- Gas-efficient for governance systems

**Linear Decay**
- Predictable for users
- Simple to audit
- Gas-efficient calculation

## Status

**Design Phase:** ✅ Complete  
**Implementation:** 🚧 In Progress  
**Testing:** 📋 Planned

---

## Author

**Edgar Velazquez**  
Smart Contract Engineer  
edgara.velazquezg@gmail.com

---

## License

MIT License - Free to use for educational purposes

