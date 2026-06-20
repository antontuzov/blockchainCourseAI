# Module 8 — Libraries & Open-Source Security

> **Part 2 · Solidity Programming**

---

## Learning Objectives

After completing this module you will be able to:

1. Understand how Solidity libraries differ from contracts
2. Use `using for` to attach library functions to types
3. Install and use OpenZeppelin contracts in Foundry
4. Recognise the most important OpenZeppelin modules (Ownable, ERC20, AccessControl)
5. Evaluate open-source code for security and versioning risks

---

## 1. What Is a Solidity Library?

A library is like a contract, but:

- **Cannot have state variables** (no storage)
- **Cannot hold ETH**
- **Cannot be destroyed**
- By default, deployed once and called via `DELEGATECALL`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

library MathLib {
    function max(uint256 a, uint256 b) internal pure returns (uint256) {
        return a >= b ? a : b;
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a <= b ? a : b;
    }

    function average(uint256 a, uint256 b) internal pure returns (uint256) {
        // (a + b) / 2 can overflow; use this instead
        return (a & b) + (a ^ b) / 2;
    }
}
```

### `internal` vs `external` Library Functions

| Modifier | Behaviour |
|----------|-----------|
| `internal` | Inlined into the calling contract's bytecode (no DELEGATECALL) |
| `public`/`external` | Deployed separately, called via DELEGATECALL |

---

## 2. `using for` — Attach Libraries to Types

```solidity
library ArrayUtils {
    function sum(uint256[] memory arr) internal pure returns (uint256 total) {
        for (uint i = 0; i < arr.length; i++) {
            total += arr[i];
        }
    }
}

contract Calculator {
    using ArrayUtils for uint256[];

    function totalPayments(uint256[] memory amounts) external pure returns (uint256) {
        return amounts.sum(); // ← Library function called on the array
    }
}
```

### File-level `using for` (Solidity ≥ 0.8.13)

```solidity
// Applies to all files that import this
using {MathLib.max, MathLib.min} for uint256 global;
```

---

## 3. Installing OpenZeppelin with Foundry

```bash
# Install OpenZeppelin contracts
forge install OpenZeppelin/openzeppelin-contracts

# This adds it to lib/ and creates a remapping
```

Add remapping to `foundry.toml`:

```toml
[profile.default]
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/"]
```

Now you can import OpenZeppelin:

```solidity
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
```

---

## 4. Key OpenZeppelin Contracts

### Ownable — Simple Ownership

```solidity
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyVault is Ownable {
    constructor() Ownable(msg.sender) {} // Pass initial owner

    function adminOnly() external onlyOwner {
        // Only the owner can call this
    }
}
```

### AccessControl — Role-Based Permissions

```solidity
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

contract Treasury is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    constructor() {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        // Only addresses with MINTER_ROLE can call this
    }
}
```

### ReentrancyGuard

```solidity
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract Vault is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // Protected against re-entrancy attacks
    }
}
```

### Pausable

```solidity
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";

contract Token is Pausable, Ownable {
    function transfer(address to, uint256 amount) external whenNotPaused {
        // Blocked when paused
    }

    function pause() external onlyOwner {
        _pause();
    }
}
```

---

## 5. OpenZeppelin's ERC-20 Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

/// @title MyToken — an ERC-20 token with minting
contract MyToken is ERC20, Ownable {
    constructor(uint256 initialSupply)
        ERC20("MyToken", "MTK")
        Ownable(msg.sender)
    {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }

    function mint(address to, uint256 amount) external onlyOwner {
        _mint(to, amount);
    }
}
```

That's it — 14 lines for a full ERC-20 token. OpenZeppelin handles:

- Balance tracking
- Allowance management
- Transfer / transferFrom
- Event emission
- Edge cases and security

---

## 6. Evaluating Open-Source Code

### Checklist Before Using Any Library

| Check | Why |
|-------|-----|
| **Audit status** | Has it been audited? By whom? |
| **Version** | Use exact versions, not `^` ranges |
| **Maintenance** | Last commit date, open issues, response time |
| **Complexity** | Simpler is better; understand what you import |
| **Gas cost** | Some abstractions add gas overhead |
| **License** | MIT, GPL, BUSL — affects your project's license |

### Version Pinning in Foundry

```bash
# Install a specific version/tag
forge install OpenZeppelin/openzeppelin-contracts@v5.0.0
```

---

## 7. Mini-Project: Build an OpenZeppelin-Based Token

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import {ERC20Pausable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";

/// @title AdvancedToken — ERC20 with burn, pause, and role-based minting
contract AdvancedToken is ERC20, ERC20Burnable, ERC20Pausable, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    constructor() ERC20("Advanced Token", "ADV") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
        _grantRole(PAUSER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }

    function pause() external onlyRole(PAUSER_ROLE) {
        _pause();
    }

    function unpause() external onlyRole(PAUSER_ROLE) {
        _unpause();
    }

    // Required override to resolve multiple inheritance of _update
    function _update(address from, address to, uint256 value)
        internal
        override(ERC20, ERC20Pausable)
    {
        super._update(from, to, value);
    }
}
```

### Test It

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {AdvancedToken} from "../src/AdvancedToken.sol";

contract AdvancedTokenTest is Test {
    AdvancedToken token;
    address alice = makeAddr("alice");

    function setUp() public {
        token = new AdvancedToken();
    }

    function test_Mint() public {
        token.mint(alice, 1000 ether);
        assertEq(token.balanceOf(alice), 1000 ether);
    }

    function test_Pause() public {
        token.mint(alice, 1000 ether);
        token.pause();
        vm.expectRevert();
        vm.prank(alice);
        token.transfer(address(1), 100 ether);
    }
}
```

---

## Checkpoint ✅

1. Install OpenZeppelin in a Foundry project and compile a contract using `ERC20`
2. Explain the difference between a `library` and a `contract`
3. What does `using ArrayUtils for uint256[]` do?
4. Why should you pin dependency versions in production?

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Using `^` version ranges in production | Pin exact versions to avoid unexpected breaking changes |
| Importing unused OpenZeppelin contracts | Each import increases deployment size — only use what you need |
| Not overriding `_update` when using ERC20Pausable | Solidity requires explicit override when two parents define the same function |
| Libraries with state | Library functions called via DELEGATECALL can modify the caller's storage, not their own |

---

## Further Reading

- [OpenZeppelin Contracts Docs](https://docs.openzeppelin.com/contracts/5.x/)
- [Solidity Docs — Libraries](https://docs.soliditylang.org/en/latest/contracts.html#libraries)
- [Foundry Dependencies](https://book.getfoundry.sh/projects/dependencies)

---

**Previous →** [Module 7: Inheritance, Interfaces & Abstract Contracts](./07-inheritance-interfaces-abstract.md)  
**Next →** [Module 9: Testing with Foundry](./09-testing-foundry.md)
