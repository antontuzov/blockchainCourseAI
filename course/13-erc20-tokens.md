# Module 13 — ERC-20 Tokens

> **Part 4 · Token Standards & DeFi Primitives**

---

## Learning Objectives

After completing this module you will be able to:

1. Explain the ERC-20 standard and why token standards matter
2. Implement a full ERC-20 token from scratch and with OpenZeppelin
3. Build an ICO crowdsale contract
4. Create a token vesting schedule
5. Understand common ERC-20 extensions (burnable, permit, votes)

---

## 1. The ERC-20 Standard

ERC-20 defines a **fungible token** interface — every token is identical and divisible.

### Required Functions

| Function | Purpose |
|----------|---------|
| `totalSupply()` | Total tokens in existence |
| `balanceOf(address)` | Token balance of an account |
| `transfer(address, uint256)` | Send tokens from caller |
| `transferFrom(address, address, uint256)` | Send tokens on behalf (requires approval) |
| `approve(address, uint256)` | Allow spender to use tokens |
| `allowance(address, address)` | Check approved amount |

### Events

| Event | When |
|-------|------|
| `Transfer(from, to, value)` | Tokens move (including mint from 0x0) |
| `Approval(owner, spender, value)` | Allowance changes |

---

## 2. ERC-20 from Scratch

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

/// @title MyERC20 — ERC-20 implementation from scratch
contract MyERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    error InsufficientBalance(address sender, uint256 balance, uint256 needed);
    error InsufficientAllowance(address spender, uint256 allowance, uint256 needed);

    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        _mint(msg.sender, _initialSupply * 10 ** decimals);
    }

    function balanceOf(address account) external view returns (uint256) {
        return _balances[account];
    }

    function allowance(address owner, address spender) external view returns (uint256) {
        return _allowances[owner][spender];
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 currentAllowance = _allowances[from][msg.sender];
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < amount) {
                revert InsufficientAllowance(msg.sender, currentAllowance, amount);
            }
            unchecked {
                _approve(from, msg.sender, currentAllowance - amount);
            }
        }
        _transfer(from, to, amount);
        return true;
    }

    function _transfer(address from, address to, uint256 amount) internal {
        require(to != address(0), "Transfer to zero address");
        uint256 fromBalance = _balances[from];
        if (fromBalance < amount) {
            revert InsufficientBalance(from, fromBalance, amount);
        }
        unchecked {
            _balances[from] = fromBalance - amount;
            _balances[to] += amount;
        }
        emit Transfer(from, to, amount);
    }

    function _mint(address to, uint256 amount) internal {
        require(to != address(0), "Mint to zero address");
        totalSupply += amount;
        unchecked {
            _balances[to] += amount;
        }
        emit Transfer(address(0), to, amount);
    }

    function _approve(address owner, address spender, uint256 amount) internal {
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
}
```

### Why `type(uint256).max` Allowance?

Some protocols (Uniswap) approve `type(uint256).max` to avoid re-approving. The check `if (currentAllowance != type(uint256).max)` ensures infinite approvals aren't decremented.

---

## 3. ERC-20 with OpenZeppelin

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC20Burnable} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import {ERC20Permit} from "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract ProToken is ERC20, ERC20Burnable, ERC20Permit, Ownable {
    uint256 public constant MAX_SUPPLY = 100_000_000 * 1e18;

    constructor()
        ERC20("Pro Token", "PRO")
        ERC20Permit("Pro Token")
        Ownable(msg.sender)
    {
        _mint(msg.sender, 10_000_000 * 1e18);
    }

    function mint(address to, uint256 amount) external onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
}
```

---

## 4. ICO Crowdsale Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

/// @title Crowdsale — buy tokens with ETH
contract Crowdsale is ReentrancyGuard {
    using SafeERC20 for IERC20;

    IERC20 public immutable token;
    address public immutable beneficiary;
    uint256 public immutable rate;        // tokens per ETH
    uint256 public immutable startTime;
    uint256 public immutable endTime;
    uint256 public immutable cap;         // max ETH to raise

    uint256 public totalRaised;
    mapping(address => uint256) public contributions;

    event TokensPurchased(address indexed buyer, uint256 ethAmount, uint256 tokenAmount);
    event SaleFinalized(uint256 totalRaised);

    error SaleNotActive();
    error CapExceeded();
    error ZeroContribution();
    error SaleNotEnded();

    constructor(
        address _token,
        address _beneficiary,
        uint256 _rate,
        uint256 _duration,
        uint256 _cap
    ) {
        token = IERC20(_token);
        beneficiary = _beneficiary;
        rate = _rate;
        startTime = block.timestamp;
        endTime = block.timestamp + _duration;
        cap = _cap;
    }

    modifier whenSaleActive() {
        if (block.timestamp < startTime || block.timestamp >= endTime) revert SaleNotActive();
        _;
    }

    /// @notice Buy tokens with ETH
    function buyTokens() external payable nonReentrant whenSaleActive {
        if (msg.value == 0) revert ZeroContribution();
        if (totalRaised + msg.value > cap) revert CapExceeded();

        uint256 tokenAmount = msg.value * rate;
        contributions[msg.sender] += msg.value;
        totalRaised += msg.value;

        token.safeTransfer(msg.sender, tokenAmount);
        emit TokensPurchased(msg.sender, msg.value, tokenAmount);
    }

    /// @notice Owner withdraws raised ETH after sale ends
    function finalize() external {
        if (block.timestamp < endTime) revert SaleNotEnded();
        (bool success,) = beneficiary.call{value: address(this).balance}("");
        require(success, "Transfer failed");
        emit SaleFinalized(totalRaised);
    }

    function isActive() external view returns (bool) {
        return block.timestamp >= startTime && block.timestamp < endTime;
    }
}
```

---

## 5. Token Vesting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

/// @title TokenVesting — release tokens over time
contract TokenVesting {
    using SafeERC20 for IERC20;

    struct VestingSchedule {
        uint256 totalAmount;
        uint256 startTime;
        uint256 cliff;      // Time until first release
        uint256 duration;   // Total vesting duration
        uint256 released;   // Already released
    }

    IERC20 public immutable token;
    mapping(address => VestingSchedule) public schedules;

    event VestingCreated(address indexed beneficiary, uint256 amount);
    event TokensReleased(address indexed beneficiary, uint256 amount);

    error NoVestingSchedule();
    error NothingToRelease();

    constructor(address _token) {
        token = IERC20(_token);
    }

    /// @notice Create a vesting schedule for a beneficiary
    function createVesting(
        address beneficiary,
        uint256 amount,
        uint256 cliff,
        uint256 duration
    ) external {
        require(cliff <= duration, "Cliff > duration");
        require(schedules[beneficiary].totalAmount == 0, "Already exists");

        schedules[beneficiary] = VestingSchedule({
            totalAmount: amount,
            startTime: block.timestamp,
            cliff: cliff,
            duration: duration,
            released: 0
        });

        token.safeTransferFrom(msg.sender, address(this), amount);
        emit VestingCreated(beneficiary, amount);
    }

    /// @notice Claim vested tokens
    function release() external {
        VestingSchedule storage schedule = schedules[msg.sender];
        if (schedule.totalAmount == 0) revert NoVestingSchedule();

        uint256 releasable = _vestedAmount(schedule) - schedule.released;
        if (releasable == 0) revert NothingToRelease();

        schedule.released += releasable;
        token.safeTransfer(msg.sender, releasable);
        emit TokensReleased(msg.sender, releasable);
    }

    function _vestedAmount(VestingSchedule storage s) internal view returns (uint256) {
        uint256 elapsed = block.timestamp - s.startTime;
        if (elapsed < s.cliff) return 0;
        if (elapsed >= s.duration) return s.totalAmount;
        return (s.totalAmount * elapsed) / s.duration;
    }

    function vestedAmount(address beneficiary) external view returns (uint256) {
        return _vestedAmount(schedules[beneficiary]);
    }
}
```

---

## 6. ERC-20 Extensions

| Extension | What it adds |
|-----------|-------------|
| `ERC20Burnable` | `burn(amount)` and `burnFrom(account, amount)` |
| `ERC20Permit` (EIP-2612) | Gasless approvals via signatures |
| `ERC20Votes` | Token-weighted governance voting |
| `ERC20Capped` | Enforces a maximum supply |
| `ERC20FlashMint` (EIP-3156) | Flash loan minting |
| `ERC20Wrapper` | Wraps an existing token (like WETH) |

---

## Checkpoint ✅

1. Implement ERC-20 from scratch (without OpenZeppelin) and test transfers
2. Deploy a token to Sepolia and buy it via the Crowdsale contract
3. Create a vesting schedule and verify the linear release calculation with a fuzz test
4. Explain the approve/transferFrom race condition and how `increaseAllowance`/`decreaseAllowance` solve it

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Front-running `approve()` | Change from 100 to 200: attacker sees pending TX, spends old 100 first |
| Not handling `fee-on-transfer` tokens | Some tokens deduct fees on transfer; use actual received amount |
| Minting to `address(0)` | Always check `to != address(0)` — tokens would be permanently lost |
| Ignoring return value of `transfer` | Use `SafeERC20` to handle non-standard tokens that don't return bool |

---

## Further Reading

- [EIP-20: ERC-20 Token Standard](https://eips.ethereum.org/EIPS/eip-20)
- [OpenZeppelin ERC20 Docs](https://docs.openzeppelin.com/contracts/5.x/erc20)
- [EIP-2612: Permit](https://eips.ethereum.org/EIPS/eip-2612)

---

**Previous →** [Module 12: Gas Optimisation](./12-gas-optimisation.md)  
**Next →** [Module 14: ERC-721 & ERC-1155 NFTs](./14-erc721-erc1155-nfts.md)
