# Module 12 — Gas Optimisation

> **Part 3 · Testing, Tooling & Deployment**

---

## Learning Objectives

After completing this module you will be able to:

1. Understand the EVM gas cost model (storage, memory, computation)
2. Use storage packing to fit multiple variables in one slot
3. Choose between `memory` and `calldata` for gas savings
4. Apply assembly (Yul) for performance-critical code paths
5. Profile gas usage with Foundry's `--gas-report` and snapshots

---

## 1. Why Gas Matters

Every operation costs gas. Users pay gas. Expensive contracts = fewer users.

| Operation | Gas Cost |
|-----------|----------|
| Cold SLOAD (read new slot) | 2,100 |
| Warm SLOAD (read cached slot) | 100 |
| SSTORE (new slot, zero→nonzero) | 20,000 |
| SSTORE (update, nonzero→nonzero) | 2,900 |
| SSTORE (nonzero→zero) | Refund of ~4,800 |
| LOG0 (event, no topics) | 375 + 8 * bytes |
| CALL (external) | 2,600 (cold) + child gas |
| CREATE2 (deploy contract) | 32,000 |

---

## 2. Storage Packing

A storage slot is **32 bytes**. Multiple small variables can share one slot if declared in the right order.

### Bad Layout (3 slots = 60,000 gas)

```solidity
contract BadPacking {
    uint8 a;      // Slot 0 (31 bytes wasted)
    uint256 b;    // Slot 1 (full slot)
    uint8 c;      // Slot 2 (31 bytes wasted)
}
```

### Good Layout (2 slots = 40,000 gas)

```solidity
contract GoodPacking {
    uint256 b;    // Slot 0 (full slot)
    uint8 a;      // Slot 1 (shares with c)
    uint8 c;      // Slot 1 (shares with a)
}
```

### Optimal Layout (1 slot = 20,000 gas)

```solidity
contract OptimalPacking {
    uint8 a;      // Slot 0 byte 0
    uint8 b;      // Slot 0 byte 1
    uint8 c;      // Slot 0 byte 2
    bool d;       // Slot 0 byte 3
    address e;    // Slot 0 bytes 4-23 (20 bytes)
    uint40 f;     // Slot 0 bytes 24-28 (5 bytes)
    // Total: 29 bytes — fits in 1 slot!
}
```

### Foundry Storage Layout Tool

```bash
forge inspect MyContract storage --pretty
```

---

## 3. Memory vs Calldata

```solidity
// Expensive: copies calldata to memory
function process(uint256[] memory data) external pure returns (uint256) {
    uint256 sum;
    for (uint i = 0; i < data.length; i++) {
        sum += data[i];
    }
    return sum;
}

// Cheap: reads directly from calldata (no copy)
function process(uint256[] calldata data) external pure returns (uint256) {
    uint256 sum;
    for (uint i = 0; i < data.length; i++) {
        sum += data[i];
    }
    return sum;
}
```

**Rule:** Always use `calldata` for external function parameters when you don't need to modify them.

---

## 4. Gas Optimisation Techniques

### Use `unchecked` for Safe Math

```solidity
// Solidity 0.8+ adds overflow checks everywhere
// If you KNOW it can't overflow, save gas:

function sum(uint256[] calldata arr) external pure returns (uint256 total) {
    uint256 len = arr.length;
    for (uint256 i; i < len;) {
        total += arr[i];
        unchecked { ++i; } // Save ~60 gas per iteration
    }
}
```

### Custom Errors Over Strings

```solidity
// Expensive: string stored in bytecode
require(balance >= amount, "Insufficient balance"); // ~200 extra gas

// Cheap: custom error (4 bytes selector)
if (balance < amount) revert InsufficientBalance(); // Saves ~200 gas
```

### Pack Function Arguments

```solidity
// Expensive: multiple storage writes
function setValues(uint256 a, uint256 b) external {
    val1 = a; // 20,000 gas
    val2 = b; // 5,000 gas (same slot update)
}

// Cheaper: single storage write using struct
struct Values { uint128 a; uint128 b; }
Values public values;
function setValues(uint128 a, uint128 b) external {
    values = Values(a, b); // 20,000 gas total
}
```

### `immutable` and `constant`

```solidity
// Expensive: SLOAD every time (2,100 gas)
uint256 public fee = 100;

// Cheap: inlined in bytecode (no storage)
uint256 public constant FEE = 100;

// Set once in constructor, then inlined
uint256 public immutable fee;
constructor(uint256 _fee) { fee = _fee; }
```

### Short-Circuit Conditions

```solidity
// Put cheaper checks first
if (msg.sender != owner) revert NotOwner(); // Cheap (SLOAD)
if (complexCondition()) revert Fail();       // Expensive

// Boolean short-circuit
if (cheapCheck() && expensiveCheck()) { } // expensiveCheck only runs if cheap passes
```

---

## 5. Assembly (Yul) — Low-Level Gas Savings

Use assembly when the Solidity compiler adds unnecessary overhead.

### Example: Read a Single Storage Slot

```solidity
function getSlot(uint256 slot) external view returns (bytes32 value) {
    assembly {
        value := sload(slot)
    }
}
```

### Example: Efficient ETH Transfer

```solidity
function withdraw(address payable to, uint256 amount) external {
    assembly {
        // call(gas, to, value, inOffset, inSize, outOffset, outSize)
        let success := call(gas(), to, amount, 0, 0, 0, 0)
        if iszero(success) {
            revert(0, 0)
        }
    }
}
```

### Example: Uniswap V2's `swap` (Simplified)

```solidity
function swap(uint amount0Out, uint amount1Out, address to) external {
    assembly {
        // Pack outputs into memory efficiently
        let freePtr := mload(0x40)
        mstore(freePtr, amount0Out)
        mstore(add(freePtr, 0x20), amount1Out)
        // ... call logic
    }
}
```

> **Warning:** Assembly bypasses safety checks. Only use it when the gas savings justify the risk. Always test thoroughly.

---

## 6. Gas Profiling with Foundry

### Gas Report

```bash
forge test --gas-report

# Output:
# ╭─────────────────────────────────────┬─────────────────┬───────┬────────┬───────┬─────────╮
# │ src/Counter.sol:Counter Contract    ┆                 ┆       ┆        ┆       ┆         │
# ╞═════════════════════════════════════╪═════════════════╪═══════╪════════╪═══════╪═════════╡
# │ Deployment Cost                     ┆                 ┆       ┆        ┆       ┆         │
# ╞╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
# │                                     ┆ 106012          ┆       ┆        ┆       ┆         │
# ├─────────────────────────────────────┼─────────────────┼───────┼────────┼───────┼─────────┤
# │ Function Name                       ┆ min             ┆ avg   ┆ median ┆ max   ┆ # calls │
# ├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┤
# │ increment                           ┆ 28334           ┆ 28334 ┆ 28334  ┆ 28334 ┆ 1       │
# │ setNumber                           ┆ 26319           ┆ 26319 ┆ 26319  ┆ 26319 ┆ 1       │
# ╰─────────────────────────────────────┴─────────────────┴───────┴────────┴───────┴─────────╯
```

### Gas Snapshots

```bash
# Save current gas usage as snapshot
forge snapshot

# Compare against snapshot after optimization
forge snapshot --diff
```

---

## 7. Mini-Project: Gas Golf Challenge

Optimize this contract to use as little gas as possible:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title GasGolf — optimize for minimum gas
contract GasGolf {
    // BEFORE (unoptimized)
    uint256 public totalDeposits;
    uint256 public totalWithdrawals;
    mapping(address => uint256) public balances;
    mapping(address => bool) public admins;
    string public name;

    function deposit() external payable {
        balances[msg.sender] = balances[msg.sender] + msg.value;
        totalDeposits = totalDeposits + msg.value;
    }

    function isAdmin(address user) external view returns (bool) {
        return admins[user];
    }
}
```

### Optimized Version

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title GasGolf — optimized version
contract GasGolfOptimized {
    // Pack: 3 uint128s + 1 bytes32 fit in 2 slots
    uint128 public totalDeposits;
    uint128 public totalWithdrawals;
    bytes32 public nameHash; // Store hash instead of string (if only checking equality)

    mapping(address => uint256) public balances;
    mapping(address => bool) public admins;

    function deposit() external payable {
        unchecked {
            balances[msg.sender] += uint128(msg.value);
            totalDeposits += uint128(msg.value);
        }
    }

    function isAdmin(address user) external view returns (bool) {
        return admins[user]; // Already optimal — single SLOAD
    }
}
```

---

## Checkpoint ✅

1. Use `forge inspect` to check the storage layout of a contract with 5 variables
2. Rewrite a function using `calldata` instead of `memory` and compare gas
3. Use `unchecked` in a for loop and measure gas savings with `--gas-report`
4. Explain when assembly is appropriate and when it's overkill

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Premature optimization | Write correct code first, then optimize hot paths |
| Assembly without testing | Yul bypasses all safety checks — test every edge case |
| Packing variables that need atomic updates | Packed variables can't be updated in a single SSTORE |
| Using `memory` for external function params | `calldata` is always cheaper when you don't need mutation |

---

## Further Reading

- [Solidity Gas Optimization Guide](https://docs.soliditylang.org/en/latest/internals/optimization.html)
- [Gas Golfing Patterns](https://github.com/Vectorized/solady)
- [EVM Gas Costs Reference](https://www.evm.codes/)
- [Foundry Gas Snapshots](https://book.getfoundry.sh/forge/gas-reports)

---

**Previous →** [Module 11: Upgradeable Contracts](./11-upgradeable-contracts.md)  
**Next →** [Module 13: ERC-20 Tokens](./13-erc20-tokens.md)
