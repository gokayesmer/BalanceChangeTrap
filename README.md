# BalanceChangeTrap


## Objective

Create a functional and deployable Ethereum trap that:

- Monitors the ETH balance of a specific wallet.
- Uses the standard `collect()` / `shouldRespond()` interface.
- Triggers a response when the balance deviation exceeds a given threshold (e.g., 0.01 ETH).
- Integrates with a separate alert contract to handle responses.

## Problem

Ethereum wallets managing critical operations like DAO treasury, DeFi protocols, or vesting mechanisms must maintain a stable balance. Unexpected changes, whether a gain or loss, could signal compromise, human error, or an exploit attempt.

## Solution

Monitor the ETH balance of a wallet across blocks and trigger a response when the balance deviation exceeds a defined threshold, alerting external systems about potential issues. This trap ensures that any significant change in the wallet’s balance is quickly detected.

## Trap Logic Summary

### Trap Contract: `BalanceChangeTrap.sol`

The trap monitors the balance of a specific wallet and compares it to the previous balance. If the change exceeds the threshold (e.g., 0.01 ETH), it triggers a response. The contract uses the standard `collect()` and `shouldRespond()` interface, and a response is generated when the threshold is exceeded.

#### Example Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ITrap {
    function collect() external view returns (bytes memory);
    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory);
}

contract BalanceChangeTrap is ITrap {
    address public constant target = 0xe4AE1F********e1c3c766aF2A19F85f73BB3; // A tracked wallet
    uint256 public constant changeThreshold = 0.01 ether;

    // Data collection: current balance
    function collect() external view override returns (bytes memory) {
        return abi.encode(target.balance);
    }

    // Checking the balance change
    function shouldRespond(bytes[] calldata data) external pure override returns (bool, bytes memory) {
        if (data.length < 2) {
            return (false, abi.encode("Insufficient data"));
        }

        uint256 current = abi.decode(data[0], (uint256));
        uint256 previous = abi.decode(data[1], (uint256));

        uint256 diff = current > previous ? current - previous : previous - current;

        if (diff >= changeThreshold) {
            return (true, abi.encode("Balance change exceeded threshold"));
        }

        return (false, bytes(""));
    }
}
```

#### Response Contract: `LogAlertReceiver.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    event Alert(string message);

    function logBalanceChangeAlert(string calldata message) external {
        emit Alert(message);
    }
}
```

## What It Solves

- Detects suspicious ETH flows from monitored wallets.
- Provides an automated alerting mechanism via event logging.
- Can be integrated with automation tools to handle responses, such as freezing funds or issuing emergency alerts in a DAO.

## Deployment & Setup Instructions

1. **Deploy the Contracts (e.g., via Foundry)**:

```bash
forge create src/BalanceChangeTrap.sol:BalanceChangeTrap   --rpc-url https://ethereum-hoodi-rpc.publicnode.com   --private-key 0x...

forge create src/LogAlertReceiver.sol:LogAlertReceiver   --rpc-url https://ethereum-hoodi-rpc.publicnode.com   --private-key 0x...
```

2. **Update `drosera.toml`**:

```toml
[traps.mytrap]
path = "out/BalanceChangeTrap.sol/BalanceChangeTrap.json"
response_contract = "<LogAlertReceiver address>"
response_function = "logBalanceChangeAlert(string)"
```

3. **Apply Changes**:

```bash
DROSERA_PRIVATE_KEY=0x... drosera apply
```

## Testing the Trap

1. Send ETH to/from the target address on the Ethereum Hoodi testnet.
2. Wait 1-3 blocks.
3. Observe logs from the Drosera operator:
   - You should see the message `ShouldRespond='true'` in the logs and Drosera dashboard if the balance change exceeds the threshold.

## Extensions & Improvements

- **Dynamic Threshold**: Allow setting a dynamic threshold via a setter function to adjust the detection sensitivity.
- **ERC-20 Support**: Extend the trap to track ERC-20 token balances in addition to native ETH.
- **Chain Multiple Traps**: Implement a unified collector to track balances across multiple wallets or tokens, allowing more complex monitoring.

This setup provides a robust mechanism for monitoring wallet balances and triggering automated actions when significant changes occur.
