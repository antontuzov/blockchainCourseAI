# Module 9 — Testing with Foundry

> **Part 3 · Testing, Tooling & Deployment**

---

## Learning Objectives

After completing this module you will be able to:

1. Write unit tests with `forge test` and the `Test` base contract
2. Use cheatcodes (`vm.*`) to manipulate blockchain state in tests
3. Write fuzz tests that discover edge cases automatically
4. Create invariant tests that verify system properties
5. Fork mainnet state for realistic integration testing

---

## 1. Test Structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Counter} from "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    // Runs before EACH test function
    function setUp() public {
        counter = new Counter();
        counter.setNumber(0);
    }

    // Test function: name must start with "test"
    function test_Increment() public {
        counter.increment();
        assertEq(counter.number(), 1);
    }

    // Test with a specific scenario
    function test_SetNumber(uint256 x) public {
        counter.setNumber(x);
        assertEq(counter.number(), x);
    }
}
```

### Running Tests

```bash
forge test                    # Run all tests
forge test -vv                # With logs
forge test -vvvv              # With traces (shows every call)
forge test --match-test test_Increment  # Run specific test
forge test --match-contract CounterTest  # Run specific contract
forge test --gas-report       # Include gas usage report
```

---

## 2. Cheatcodes — `vm.*`

Cheatcodes let you bend the rules of the EVM for testing purposes.

### Time Manipulation

```solidity
function test_StakingReward() public {
    stakeContract.deposit{value: 1 ether}();

    // Fast forward 30 days
    vm.warp(block.timestamp + 30 days);

    uint256 reward = stakeContract.pendingReward(address(this));
    assertGt(reward, 0);
}

// Mine blocks
vm.roll(block.number + 100); // Advance 100 blocks
```

### Impersonate Addresses (`prank`)

```solidity
function test_OnlyOwner() public {
    address alice = makeAddr("alice");

    // Next call will be FROM alice
    vm.prank(alice);
    vm.expectRevert(Ownable.NotOwner.selector);
    contract.adminFunction();
}

// Start prank — ALL subsequent calls are from this address
vm.startPrank(alice);
contract.deposit{value: 1 ether}();
contract.withdraw();
vm.stopPrank();
```

### Set Balances and Storage

```solidity
// Give an address ETH
vm.deal(alice, 100 ether);

// Set a storage slot directly
vm.store(address(contract), bytes32(uint256(0)), bytes32(uint256(999)));
// Slot 0 now contains 999
```

### Common Cheatcodes

| Cheatcode | Purpose |
|-----------|---------|
| `vm.prank(addr)` | Next call from `addr` |
| `vm.startPrank(addr)` | All calls from `addr` |
| `vm.warp(timestamp)` | Set block timestamp |
| `vm.roll(blockNum)` | Set block number |
| `vm.deal(addr, amount)` | Set ETH balance |
| `vm.expectRevert()` | Expect next call to revert |
| `vm.expectEmit()` | Expect an event |
| `vm.store(addr, slot, val)` | Set storage slot |
| `vm.load(addr, slot)` | Read storage slot |
| `vm.label(addr, "name")` | Label address in traces |
| `vm.snapshot()` | Snapshot state (can revert to) |
| `vm.makePersistent(addr)` | Keep state across `vm.rollFork` |

---

## 3. Testing Reverts and Events

### Testing Reverts

```solidity
function test_RevertOnZeroDeposit() public {
    vm.expectRevert(StakingContract.InsufficientStake.selector);
    staking.stake{value: 0}();
}

function test_RevertWithCustomError() public {
    vm.expectRevert(
        abi.encodeWithSelector(
            Token.InsufficientBalance.selector,
            0,    // available
            100   // required
        )
    );
    token.transfer(alice, 100);
}
```

### Testing Events

```solidity
function test_EmitsTransfer() public {
    vm.expectEmit(true, true, false, true);
    emit IERC20.Transfer(address(this), alice, 100);

    token.transfer(alice, 100);
}
```

---

## 4. Fuzz Testing

Fuzz testing runs your test with **random inputs** to discover edge cases.

```solidity
// Forge will call this with random uint256 values
function testFuzz_SetNumber(uint256 x) public {
    counter.setNumber(x);
    assertEq(counter.number(), x);
}

// Bound fuzz inputs to a reasonable range
function testFuzz_Deposit(uint96 amount) public {
    amount = uint96(bound(amount, 1 wei, 1000 ether));
    vm.deal(alice, amount);

    vm.prank(alice);
    staking.deposit{value: amount}();

    assertEq(staking.balances(alice), amount);
}
```

### Fuzz Configuration

```toml
# foundry.toml
[profile.default.fuzz]
runs = 1000          # Number of random inputs (default: 256)
max_test_rejects = 65536  # Max rejects before failure
```

---

## 5. Invariant Testing

Invariant tests verify that certain properties **always hold**, no matter what sequence of actions is performed.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {Vault} from "../src/Vault.sol";

contract VaultInvariantTest is Test {
    Vault public vault;

    function setUp() public {
        vault = new Vault();
    }

    // Invariant: total deposits >= total withdrawals
    // Property-based: Forge calls random functions in random order
    function invariant_totalSupplyMatchesDeposits() public {
        assertGe(
            vault.totalDeposited(),
            vault.totalWithdrawn(),
            "Deposits must be >= withdrawals"
        );
    }

    // Invariant: contract balance >= sum of all user balances
    function invariant_balanceExceedsLiabilities() public {
        assertGe(
            address(vault).balance,
            vault.totalDeposited() - vault.totalWithdrawn()
        );
    }
}
```

### Invariant Configuration

```toml
[profile.default.invariant]
runs = 256
depth = 15           # Max calls per sequence
fail_on_revert = false  # Don't fail on expected reverts
```

---

## 6. Fork Testing

Test against real mainnet contracts:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract UniswapForkTest is Test {
    // Mainnet addresses
    IERC20 constant USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    address constant USDC_WHALE = 0x55FE002aefF02F77364de339a1292923A15844B8;

    function setUp() public {
        // Fork at a specific block
        vm.createSelectFork("mainnet", 19_000_000);
    }

    function test_WhaleHasUSDC() public view {
        assertGt(USDC.balanceOf(USDC_WHALE), 1_000_000 * 1e6);
    }

    function test_TransferUSDC() public {
        vm.prank(USDC_WHALE);
        USDC.transfer(address(this), 1000 * 1e6);
        assertEq(USDC.balanceOf(address(this)), 1000 * 1e6);
    }
}
```

```bash
# Run fork tests (needs RPC URL)
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY forge test --fork-url $MAINNET_RPC_URL
```

---

## 7. Test Helpers

### `makeAddr` — Named Test Addresses

```solidity
address alice = makeAddr("alice");
address bob = makeAddr("bob");
// These show up as "alice" and "bob" in traces (much easier to debug)
```

### `bound` — Constrain Fuzz Inputs

```solidity
function testFuzz(uint256 x) public {
    x = bound(x, 1, 1000); // x is now between 1 and 1000
}
```

### `deal` — Set Token Balances

```solidity
// Works for ETH
vm.deal(alice, 100 ether);

// Works for ERC-20 tokens (uses vm.store internally)
deal(address(usdc), alice, 1_000_000 * 1e6);
```

---

## 8. Mini-Project: Test Suite for Staking Contract

Write a comprehensive test suite for the staking contract from Module 6:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {SimpleStaking} from "../src/SimpleStaking.sol";

contract SimpleStakingTest is Test {
    SimpleStaking staking;
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");

    function setUp() public {
        staking = new SimpleStaking();
        vm.deal(alice, 100 ether);
        vm.deal(bob, 100 ether);
    }

    function test_Stake() public {
        vm.prank(alice);
        staking.stake{value: 1 ether}();
        assertEq(staking.totalStaked(), 1 ether);
    }

    function test_RevertOnZeroStake() public {
        vm.prank(alice);
        vm.expectRevert(SimpleStaking.InsufficientStake.selector);
        staking.stake{value: 0}();
    }

    function test_RevertOnDoubleStake() public {
        vm.startPrank(alice);
        staking.stake{value: 1 ether}();
        vm.expectRevert(SimpleStaking.AlreadyStaked.selector);
        staking.stake{value: 2 ether}();
        vm.stopPrank();
    }

    function test_WithdrawWithReward() public {
        vm.prank(alice);
        staking.stake{value: 10 ether}();

        vm.warp(block.timestamp + 365 days);

        uint256 balBefore = alice.balance;
        vm.prank(alice);
        staking.withdraw();

        uint256 received = alice.balance - balBefore;
        assertGt(received, 10 ether); // 10 ETH + reward
    }

    function testFuzz_StakeAmount(uint96 amount) public {
        amount = uint96(bound(amount, 1 wei, 50 ether));
        vm.deal(alice, amount);
        vm.prank(alice);
        staking.stake{value: amount}();
        assertEq(staking.totalStaked(), amount);
    }

    receive() external payable {}
}
```

---

## Checkpoint ✅

1. Write a test that expects a specific event to be emitted
2. Create a fuzz test for a function that takes an `address` parameter
3. Write an invariant test for a token contract: "sum of all balances == totalSupply"
4. Fork mainnet and test an interaction with a real DeFi protocol

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Forgetting `vm.startPrank` vs `vm.prank` | `prank` = next call only; `startPrank` = all subsequent calls |
| Fuzz test passing with small inputs only | Use `bound()` to constrain to meaningful ranges |
| Not labeling addresses | Debug traces show `0x1234...` instead of `"alice"` — hard to read |
| Invariant test too strict | Set `fail_on_revert = false` for expected reverts |

---

## Further Reading

- [Foundry Book — Testing](https://book.getfoundry.sh/forge/testing)
- [Foundry Cheatcodes Reference](https://book.getfoundry.sh/cheatcodes/)
- [Invariant Testing Guide](https://book.getfoundry.sh/forge/invariant-testing)

---

**Previous →** [Module 8: Libraries & OpenZeppelin](./08-libraries-openzeppelin.md)  
**Next →** [Module 10: Deploying & Verifying Contracts](./10-deploying-verifying.md)
