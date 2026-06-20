# Module 10 — Deploying & Verifying Contracts

> **Part 3 · Testing, Tooling & Deployment**

---

## Learning Objectives

After completing this module you will be able to:

1. Write a deployment script using Foundry's `Script`
2. Deploy to local (anvil), testnet (Sepolia), and mainnet
3. Verify contract source code on Etherscan
4. Manage deployment environments and secrets
5. Automate post-deployment tasks (fund, configure, interact)

---

## 1. Deployment Scripts in Foundry

Foundry uses Solidity itself for deployment scripts — no JavaScript needed.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console} from "forge-std/Script.sol";
import {Counter} from "../src/Counter.sol";

contract DeployCounter is Script {
    function run() external {
        // Read deployer private key from env
        uint256 deployerKey = vm.envUint("PRIVATE_KEY");

        // Start broadcasting transactions
        vm.startBroadcast(deployerKey);

        // Deploy the contract
        Counter counter = new Counter();
        console.log("Counter deployed at:", address(counter));

        // Set initial number
        counter.setNumber(42);
        console.log("Initial number set to 42");

        vm.stopBroadcast();
    }
}
```

### Run the Script

```bash
# Deploy to local anvil
forge script script/DeployCounter.s.sol:DeployCounter \
  --rpc-url http://localhost:8545 \
  --broadcast

# Deploy to Sepolia testnet
forge script script/DeployCounter.s.sol:DeployCounter \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify

# Deploy to mainnet (CAREFUL — real ETH!)
forge script script/DeployCounter.s.sol:DeployCounter \
  --rpc-url $MAINNET_RPC_URL \
  --broadcast \
  --verify
```

---

## 2. Multi-Contract Deployment

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console} from "forge-std/Script.sol";
import {MyToken} from "../src/MyToken.sol";
import {TokenVault} from "../src/TokenVault.sol";

contract DeployProtocol is Script {
    function run() external {
        uint256 deployerKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerKey);

        vm.startBroadcast(deployerKey);

        // 1. Deploy token
        MyToken token = new MyToken("Protocol Token", "PRT", 1_000_000);
        console.log("Token:", address(token));

        // 2. Deploy vault
        TokenVault vault = new TokenVault(address(token));
        console.log("Vault:", address(vault));

        // 3. Configure: grant vault minter role
        token.grantRole(token.MINTER_ROLE(), address(vault));

        // 4. Transfer ownership
        token.transferOwnership(deployer);
        vault.transferOwnership(deployer);

        vm.stopBroadcast();

        // Log deployment info
        console.log("---");
        console.log("Deployer:", deployer);
        console.log("Chain ID:", block.chainid);
    }
}
```

---

## 3. Contract Verification

Verification publishes your source code on Etherscan so anyone can audit it.

### Automatic Verification

```bash
forge script script/DeployCounter.s.sol:DeployCounter \
  --rpc-url $SEPOLIA_RPC_URL \
  --broadcast \
  --verify \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

### Manual Verification

```bash
# If auto-verification fails, verify manually
forge verify-contract \
  0xYourContractAddress \
  src/Counter.sol:Counter \
  --chain sepolia \
  --etherscan-api-key $ETHERSCAN_API_KEY

# With constructor arguments
forge verify-contract \
  0xYourContractAddress \
  src/MyToken.sol:MyToken \
  --constructor-args $(cast abi-encode "constructor(string,string,uint256)" "MyToken" "MTK" 1000000) \
  --chain sepolia \
  --etherscan-api-key $ETHERSCAN_API_KEY
```

### Getting an Etherscan API Key

1. Create an account at [etherscan.io](https://etherscan.io)
2. Go to **API Keys** → **Create New API Key**
3. Store it: `export ETHERSCAN_API_KEY="your_key"`

---

## 4. Environment Management

### `.env` File (NEVER commit this)

```bash
# .env
PRIVATE_KEY=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/YOUR_KEY
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
ETHERSCAN_API_KEY=YOUR_KEY
```

```bash
# Load environment variables
source .env
```

### `.gitignore`

```
.env
out/
cache/
broadcast/
```

### `foundry.toml` RPC Endpoints

```toml
[rpc_endpoints]
localhost = "http://localhost:8545"
sepolia = "${SEPOLIA_RPC_URL}"
mainnet = "${MAINNET_RPC_URL}"
arbitrum = "https://arb1.arbitrum.io/rpc"
optimism = "https://mainnet.optimism.io"

[etherscan]
sepolia = { key = "${ETHERSCAN_API_KEY}" }
mainnet = { key = "${ETHERSCAN_API_KEY}" }
```

---

## 5. Deploy Helper Script

A reusable pattern with network detection:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Script, console} from "forge-std/Script.sol";

abstract contract BaseScript is Script {
    uint256 internal deployerKey;
    address internal deployer;

    // Different multicall addresses per chain
    function _getChainId() internal view returns (uint256) {
        return block.chainid;
    }

    function _isLocal() internal view returns (bool) {
        uint256 chainId = _getChainId();
        return chainId == 31337; // anvil
    }

    function _setupDeployer() internal {
        if (_isLocal()) {
            // Use anvil default key for local
            deployerKey = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80;
        } else {
            deployerKey = vm.envUint("PRIVATE_KEY");
        }
        deployer = vm.addr(deployerKey);
        console.log("Deployer:", deployer);
        console.log("Chain ID:", _getChainId());
    }
}
```

---

## 6. Post-Deployment Interaction

```bash
# Call a read function
cast call 0xContractAddress "totalSupply()(uint256)" \
  --rpc-url $SEPOLIA_RPC_URL

# Send a transaction
cast send 0xContractAddress "transfer(address,uint256)" \
  0xRecipient 1000000000000000000 \
  --private-key $PRIVATE_KEY \
  --rpc-url $SEPOLIA_RPC_URL

# Get contract ABI
cast interface 0xContractAddress \
  --rpc-url $SEPOLIA_RPC_URL
```

---

## 7. Mini-Project: Deploy Token to Sepolia

```bash
# 1. Setup
forge init deploy_demo && cd deploy_demo
forge install OpenZeppelin/openzeppelin-contracts

# 2. Configure foundry.toml with remappings and RPC endpoints

# 3. Create MyToken contract (from Module 8)

# 4. Create deployment script
# script/DeployToken.s.sol

# 5. Get Sepolia testnet ETH from a faucet

# 6. Deploy + Verify
source .env
forge script script/DeployToken.s.sol:DeployToken \
  --rpc-url sepolia \
  --broadcast \
  --verify \
  -vvvv

# 7. Interact
cast call $TOKEN_ADDRESS "name()(string)" --rpc-url sepolia
cast call $TOKEN_ADDRESS "totalSupply()(uint256)" --rpc-url sepolia
```

---

## Checkpoint ✅

1. Deploy a contract to anvil using a Foundry script
2. Deploy the same contract to Sepolia and verify it on Etherscan
3. Use `cast` to call a read function on your deployed Sepolia contract
4. Write a deployment script that deploys two contracts and configures one to reference the other

---

## Common Pitfalls

| Pitfall | Clarification |
|---------|---------------|
| Forgetting `--broadcast` | Script runs in simulation mode only — no transactions sent |
| Verification fails with "compiler version mismatch" | Ensure `solc_version` in `foundry.toml` matches your pragma |
| Private key leaked in git history | Always use `.env` + `.gitignore`; rotate key if exposed |
| Running out of testnet ETH mid-deployment | Fund your deployer address with extra ETH before deploying |

---

## Further Reading

- [Foundry Book — Scripting](https://book.getfoundry.sh/tutorials/solidity-scripting)
- [Etherscan API Documentation](https://docs.etherscan.io/)
- [Sourcify — Decentralized Verification](https://sourcify.dev/)

---

**Previous →** [Module 9: Testing with Foundry](./09-testing-foundry.md)  
**Next →** [Module 11: Upgradeable Contracts](./11-upgradeable-contracts.md)
