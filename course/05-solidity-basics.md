# Module 5 — Solidity Basics

> **Part 2 · Solidity Programming**

---

## Learning Objectives

After completing this module you will be able to:

1. Write and compile a basic Solidity contract
2. Use value types (`uint256`, `address`, `bool`, `bytes32`) and reference types (`arrays`, `mappings`, `structs`)
3. Understand storage vs. memory vs. calldata
4. Declare state variables with correct visibility
5. Build a simple voting contract from scratch

---

## 1. Anatomy of a Solidity File

```solidity
// SPDX-License-Identifier: MIT          // 1. License identifier (required)
pragma solidity ^0.8.24;                  // 2. Compiler version

/// @title SimpleStorage                  // 3. NatSpec documentation
/// @notice Stores and retrieves a number
contract SimpleStorage {
    uint256 private storedNumber;         // 4. State variable

    constructor(uint256 _initial) {       // 5. Constructor
        storedNumber = _initial;
    }

    function set(uint256 _n) external {   // 6. Function
        storedNumber = _n;
    }

    function get() external view returns (uint256) {
        return storedNumber;
    }
}
```

---

## 2. Value Types

| Type | Size | Example | Notes |
|------|------|---------|-------|
| `bool` | 1 byte (but padded to 32) | `true`, `false` | |
| `uint256` | 32 bytes | `0` to `2^256 - 1` | Default unsigned integer |
| `uint8` | 1 byte | `0` to `255` | Useful for small counters |
| `int256` | 32 bytes | `-2^255` to `2^255 - 1` | Signed integer |
| `address` | 20 bytes | `0xf39F...2266` | Ethereum address |
| `address payable` | 20 bytes | Same + `.transfer()` | Can receive ETH |
| `bytes32` | 32 bytes | `0xabcd...` | Fixed-size byte array |
| `bytes` | Dynamic | `hex"abcd"` | Dynamic byte array |
| `string` | Dynamic | `"hello"` | UTF-8 string (stored as bytes) |
| `enum` | 1 byte | `enum Status { Active, Paused }` | Custom named values |

### Address Special Features

```solidity
address payable alice = payable(0x70997970C51812dc3A010C7d01b50e0d17dc79C8);

// Properties
alice.balance;        // ETH balance in wei

// Methods (payable only)
alice.transfer(1 ether);     // Sends ETH, reverts on failure (gas: 2300)
bool ok = alice.send(1 ether); // Same but returns bool instead of reverting
(bool success,) = alice.call{value: 1 ether}(""); // Recommended: forward all gas
require(success, "Transfer failed");
```

> **Best practice:** Always prefer `call{value: ...}("")` over `transfer()` and `send()`. `transfer()` has a 2300 gas limit that can break when gas costs change.

---

## 3. Reference Types

### Arrays

```solidity
contract ArrayDemo {
    // Fixed-size array (stored in storage)
    uint256[5] public fixedArray;

    // Dynamic array
    uint256[] public dynamicArray;

    function demo() external {
        dynamicArray.push(10);       // Add element
        dynamicArray.push(20);
        dynamicArray.push(30);

        dynamicArray[0];             // Read: 10
        dynamicArray.length;         // 3

        dynamicArray.pop();          // Remove last: [10, 20]

        // Delete element (leaves a gap — doesn't shift!)
        delete dynamicArray[0];      // [0, 20]
    }
}
```

> **Gotcha:** `delete array[i]` sets the element to its default value but does NOT shrink the array. To remove and shift, swap with last element then pop.

### Mappings

```solidity
contract MappingDemo {
    // mapping(keyType => valueType)
    mapping(address => uint256) public balances;
    mapping(address => mapping(address => uint256)) public allowances; // nested

    function setBalance(address _user, uint256 _amount) external {
        balances[_user] = _amount;
    }

    function getBalance(address _user) external view returns (uint256) {
        return balances[_user]; // Returns 0 if never set (default value)
    }
}
```

**Key rules:**
- Mappings have no `.length` — you can't iterate them
- Keys are not stored — only `keccak256(key)` → storage slot
- To make a mapping iterable, keep a separate array of keys

### Structs

```solidity
contract StructDemo {
    struct User {
        address addr;
        uint256 balance;
        bool isActive;
        uint256[] deposits; // nested dynamic array
    }

    mapping(address => User) public users;

    function createUser(address _addr) external {
        // Method 1: positional (fragile — order matters)
        // users[_addr] = User(_addr, 0, true, new uint256[](0));

        // Method 2: named (preferred — readable and reorderable)
        users[_addr] = User({
            addr: _addr,
            balance: 0,
            isActive: true,
            deposits: new uint256[](0)
        });
    }

    function addDeposit(address _addr, uint256 _amount) external {
        User storage user = users[_addr]; // 'storage' = reference, modifies original
        user.balance += _amount;
        user.deposits.push(_amount);
    }
}
```

---

## 4. Data Locations

Every variable lives in one of three places:

| Location | Lifetime | Cost | When to use |
|----------|----------|------|-------------|
| `storage` | Permanent (on-chain) | Expensive (20k gas/write) | State variables |
| `memory` | Function execution only | Cheap | Function parameters, temp variables |
| `calldata` | Function execution, read-only | Cheapest | External function parameters |

```solidity
contract DataLocation {
    uint256[] public storedArray; // storage

    function processData(uint256[] calldata input) external {
        // calldata: read-only, cheapest for external functions
        uint256 sum = 0;
        for (uint i = 0; i < input.length; i++) {
            sum += input[i];
        }

        // memory: mutable copy
        uint256[] memory temp = new uint256[](input.length);
        for (uint i = 0; i < input.length; i++) {
            temp[i] = input[i] * 2;
        }

        // storage: persist to chain
        storedArray = temp;
    }
}
```

> **Rule of thumb:** Use `calldata` for external function inputs (cheapest), `memory` for temporary computation, and `storage` only when you need to persist.

---

## 5. Constants and Immutable

```solidity
contract Constants {
    // Known at compile time — no storage slot, cheapest
    uint256 public constant MAX_SUPPLY = 1_000_000;
    address public constant OWNER = 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266;

    // Set once in constructor — stored in bytecode, not a storage slot
    uint256 public immutable deployTime;
    address public immutable deployer;

    constructor() {
        deployTime = block.timestamp;
        deployer = msg.sender;
    }
}
```

**Gas savings:** `constant` and `immutable` values are inlined into the bytecode — no `SLOAD` needed. Always prefer these for values that don't change.

---

## 6. Type Conversions

```solidity
contract Conversions {
    function demo() external pure returns (uint256, address, bytes32) {
        // uint → address
        uint256 n = 123;
        address a = address(uint160(n));

        // address → uint
        uint256 back = uint256(uint160(a));

        // bytes32 → uint256
        bytes32 b = bytes32(uint256(42));
        uint256 val = uint256(b);

        // uint256 → smaller type (explicit cast needed)
        uint256 big = 300;
        uint8 small = uint8(big); // 300 % 256 = 44 (OVERFLOW!)

        return (val, a, b);
    }
}
```

> **Danger:** Casting to a smaller type silently truncates. Always check bounds before downcasting.

---

## 7. Mini-Project: Voting Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title SimpleVoting — vote on proposals
/// @notice Each address can vote once per proposal
contract SimpleVoting {
    struct Proposal {
        string description;
        uint256 voteCount;
    }

    address public chairperson;
    Proposal[] public proposals;
    mapping(address => mapping(uint256 => bool)) public hasVoted;

    error NotChairperson();
    error AlreadyVoted();
    error InvalidProposal();

    event ProposalCreated(uint256 indexed proposalId, string description);
    event Voted(address indexed voter, uint256 indexed proposalId);

    constructor() {
        chairperson = msg.sender;
    }

    function createProposal(string calldata _description) external {
        if (msg.sender != chairperson) revert NotChairperson();
        proposals.push(Proposal({description: _description, voteCount: 0}));
        emit ProposalCreated(proposals.length - 1, _description);
    }

    function vote(uint256 _proposalId) external {
        if (_proposalId >= proposals.length) revert InvalidProposal();
        if (hasVoted[msg.sender][_proposalId]) revert AlreadyVoted();

        hasVoted[msg.sender][_proposalId] = true;
        proposals[_proposalId].voteCount++;
        emit Voted(msg.sender, _proposalId);
    }

    function getProposalCount() external view returns (uint256) {
        return proposals.length;
    }
}
```

### Test It

```bash
forge build
forge test -vv
```

---

## Checkpoint ✅

1. Create a `TodoList` contract with `create(string)`, `toggle(uint256)`, and `getAll()` functions
2. Explain the difference between `storage`, `memory`, and `calldata` in your own words
3. What is the storage cost difference between a `constant` and a regular state variable?

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Using `string` for fixed-length data | Use `bytes32` for strings ≤ 32 chars — saves gas |
| Iterating over a mapping | Mappings aren't iterable; maintain a parallel array of keys |
| Forgetting `payable` | `address` can't receive ETH; use `address payable` |
| Unchecked downcasting | `uint256(300)` → `uint8` silently becomes 44 |

---

## Further Reading

- [Solidity Docs — Types](https://docs.soliditylang.org/en/latest/types.html)
- [Solidity by Example](https://solidity-by-example.org/)
- [Foundry Book — Creating a New Project](https://book.getfoundry.sh/projects/creating-a-new-project)

---

**Previous →** [Module 4: Dev Environment Setup](./04-dev-environment-setup.md)  
**Next →** [Module 6: Functions, Modifiers, Events & Errors](./06-functions-modifiers-events-errors.md)
