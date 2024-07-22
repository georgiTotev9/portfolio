# Cyfrin Updraft Boss Bridge Audit

## High

### [H-1] `withdrawTokensToL1` and `sendToL1` functions are vulnerable to replay attacks

**Description:** The `sendToL1` and `withdrawTokensToL1` functions allow for the withdrawal of tokens from `L2` to `L1` based on a provided signature. However, the lack of nonce verification exposes the contract to replay attacks. The proof of concept illustrates how an attacker, having successfully withdrawn tokens once, can reuse the same valid signature to execute the function multiple times. This results in the unauthorized withdrawal of tokens, as the contract does not validate whether the same signature has been used before.

**Impact:** The impact of this vulnerability is severe, as it allows an attacker to repeatedly execute token withdrawals with the same signature, potentially draining the contract's token balance and causing financial harm.

**Proof of Concept:**

```solidity
function testMultiCallWithSameSignature() public {
    uint256 depositAmount = 10e18;
    uint256 withdrawAmount = 1e18;

    // User deposit tokens on L1
    vm.startPrank(user);
    token.approve(address(tokenBridge), depositAmount);
    tokenBridge.depositTokensToL2(user, userInL2, depositAmount);
    vm.stopPrank();

    // Operator signs the message to withdraw tokens
    bytes memory message = _getTokenWithdrawalMessage(hacker, withdrawAmount);
    (uint8 v, bytes32 r, bytes32 s) = _signMessage(message, operator.key);

    // Hacker can withdraw tokens multiple times with the same signature
    vm.startPrank(hacker);
    tokenBridge.withdrawTokensToL1(hacker, withdrawAmount, v, r, s);
    tokenBridge.withdrawTokensToL1(hacker, withdrawAmount, v, r, s);
    tokenBridge.withdrawTokensToL1(hacker, withdrawAmount, v, r, s);
    tokenBridge.withdrawTokensToL1(hacker, withdrawAmount, v, r, s);
    vm.stopPrank();

    assertEq(token.balanceOf(hacker), (withdrawAmount * 4));
}
```

**Recommended Mitigation:** Implement Nonce Verification: Introduce a nonce parameter in the function signature and maintain a nonce registry for each signer. Ensure that the provided nonce is greater than the previously used nonce for the same signer. Also, to prevent the same signature from being used between `L1` and `L2`, it is recommended to add the `chainId` parameter within the signature.

### [H-2] Used `CREATE` opcode in `TokenFactory::deployToken` is not compatible with zkSync Era

**Description:** In the current code, developers are using `CREATE`, but in zkSync Era, `CREATE` for arbitrary bytecode is not available, so a revert occurs in the `deployToken` process.

<details><summary>Code</summary>

```solidity
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }
}
```
</details>

**Impact:** The protocol will not work on zkSync, and that will break all the business logic.

**Recommended Mitigation:** Follow the instructions stated in zkSync docs [here](https://era.zksync.io/docs/reference/architecture/differences-with-ethereum.html#evm-instructions).

### [H-3] Steal any tokens approved to L1Bridge

**Description:** The `depositTokensToL2(...)` function takes in `from`, `l2Recipient`, and `amount` parameters. In the function body, a `safeTransferFrom(...)` call is made to the token, where `amount` funds are transferred from the specified `from` address to the vault. An attacker can specify this `from` address as a function argument. A `Deposit` event is then emitted by the contract indicating a successful deposit, where the `l2Recipient` is then permitted to mint the token on the L2. As a result, an attacker can specify any `from` address for a user that has previously approved `token` (and other tokens when they are added) to be spent by the `L1BossBridge` contract, and set themselves as the `l2Recipient`. In doing so, they steal funds of the user on the L1 side and mint it to the attacker on the L2 side.

**Impact:** Loss of funds for any user approving the `L1BossBridge` to spend funds on the L1.

**Proof of Concept:**

#### Test

```solidity
function test_TokenApprovalThief() public {
    // Create users
    address alice = vm.addr(1);
    vm.label(alice, "Alice");

    address bob = vm.addr(2);
    vm.label(bob, "Bob");

    // Distribute tokens
    deal(address(token), alice, 10e18);

    console2.log("Token balance alice (before):", token.balanceOf(alice));
    console2.log("Token balance bob (before):", token.balanceOf(bob));

    // Alice approves bridge
    vm.prank(alice);
    token.approve(address(tokenBridge), type(uint256).max);

    // Bob calls depositTokensToL2 with alice as from
    vm.prank(bob);
    tokenBridge.depositTokensToL2(alice, bob, 10e18);

    console2.log("Token balance alice (after):", token.balanceOf(alice));
}
```
**Recommended Mitigation:** Remove the `from` parameter and change `token.safeTransferFrom(from, address(vault), amount);` to `token.safeTransferFrom(msg.sender, address(vault), amount);`.

### [H-4] `L1Token` contract deployment from `TokenFactory` locks tokens forever

**Description:** `TokenFactory::deployToken` deploys `L1Token` contracts, but the `L1Token` mints initial supply to `msg.sender`, in this case, the `TokenFactory` contract itself. After deployment, there is no way to either transfer out these tokens or mint new ones, as the holder of the tokens, `TokenFactory`, has no functions for this, and is also not an upgradeable contract, so all token supply is locked forever.

**Impact:** Using this token factory to deploy tokens will result in unusable tokens, and no transfers can be made.

**Recommended Mitigation:** Tokens should be minted to a receiver different from `msg.sender`.

### [H-5] By sending any amount, attacker can withdraw more funds than deposited

**Description:** The `withdrawTokensToL1()` function has no validation on the withdrawal `amount` being the same as the deposited `amount`. As such, any user can drain the entire vault. Note that even the docs state that the `operator`, before signing a message, only checks that the user had made a successful deposit; nothing about the deposit amount.

**Impact:** Attacker can steal all the funds from the vault.

**Proof of Concept:** 
Steps:
1. Attacker deposits 1 wei (or 0 wei) into the L2 bridge.
2. Attacker crafts and encodes a malicious message and submits it to the `operator` to be signed. The malicious message has the `amount` field set to a high value, like the total funds available in the `vault`.
3. Since the attacker had deposited 1 wei, operator approves & signs the message, not knowing the contents of it since it is encoded.
4. Attacker calls `withdrawTokensToL1()`.
5. All vault's funds are transferred to the attacker.

**Recommended Mitigation:** Add a mapping that keeps track of the amount deposited by an address inside the function `depositTokensToL2()`, and validate that inside `withdrawTokensToL1()`.

## Low

### [L-1] Missing events

**Description:** `L1BossBridge::withdrawToL1`, `L1BossBridge::sendToL1`, and `L1BossBridge::setSigner` do not emit events. Therefore, changes to the signers and withdrawals are not able to be viewed off-chain.

**Impact:** When the state is initialized or modified, an event needs to be emitted. Any state that is initialized or modified without an event being emitted is not visible off-chain. This means that any off-chain service is not able to view changes. For example, the key operators might look at the events to see how many signers had been set or withdrawals that have taken place.

**Recommended Mitigation:** Emit events for state-changing transactions.

## Informational

### [I-1] `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>2 Found Instances</summary>

- Found in src/TokenFactory.sol [Line: 23]
```solidity
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
```

- Found in src/TokenFactory.sol [Line: 31]
```solidity
function getTokenAddressFromSymbol(string memory symbol) public view returns (address addr) {
```

</details>

### [I-2] Non-mutated `storage` variables could be marked as `constant` or `immutable`

Instead of leaving variables that don't need to be mutated as `storage`, consider marking them as `constant` or `immutable` to save gas.

<details><summary>2 Found Instances</summary>

- Found in src/L1Vault.sol [Line: 13]
```solidity
contract L1Vault is Ownable {
    IERC20 public token;
```

- Found in src/L1BossBridge.sol [Line: 30]
```solidity
contract L1BossBridge is Ownable, Pausable, ReentrancyGuard {
    using SafeERC20 for IERC20;

    uint256 public DEPOSIT_LIMIT = 100_000 ether;
```

</details>

### [I-3] Emitting event `Deposit` in `L1BossBridge::depositTokensToL2` should be moved above `token.safeTransferFrom()` to follow CEI

### [I-4] `L1Vault::approveTo` should check `token.approve()` return value
