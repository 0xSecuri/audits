# Findings Summary

- High : 2
- Medium: 2

| ID            | Title                                                                     | Severity |
| ------------- | :------------------------------------------------------------------------ | :------: |
| [H-01](#H-01) | Users won't be able to withdraw their Kerosene tokens even from the unbounded kerosine vault |   High   |
| [H-02](#H-02) | Kerosene USD Value Calculation Flaw Causes Misrepresentation of Kerosene Value and Collateral Ratios |   High   |
| [M-01](#M-01) | Flaw in Kerosene Asset Price Calculations Leads to Breakage in Other Functions |   Medium   |
| [M-02](#M-02) | Kerosene Price Manipulation Vulnerability Leading to Protocol Instability |   Medium   |

# Detailed Findings

## <a id='H-01'></a>H-01 Users won't be able to withdraw their Kerosene tokens even from the unbounded kerosine vault 

 # Lines of code

https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L134-L153


# Vulnerability details

## Impact

The Kerosene token can be deposited either in bounded or in unbounded Kerosene vault. If it is deposited in the unbounded vault, users should be able to withdraw their deposited tokens. However, there is a flaw in the implementation which will result in users being completely unable to withdraw their Kerosene tokens from the protocol.

## Proof of Concept

In the current implementation, in order to withdraw Kerosene tokens, the user should call the `VaultManagerV2::withdraw()` function by passing as an argument the address of the unbounded Kerosene vault to which they have deposited their Kerosene tokens. However, the unbounded Kerosene vault does not have an `oracle`, and because of that the [withdraw()](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L134-L153) function simply reverts. To confirm this, please follow the steps below:

1. In the test folder create a new file called `VaultManagerV2Test.t.sol`
2. Copy the following PoC content and paste it there

```solidity
pragma solidity =0.8.17;

import {VaultManagerTestHelper} from "./VaultManagerHelper.t.sol";
import {Test, console} from "forge-std/Test.sol";
import {Kerosine} from "../src/staking/Kerosine.sol";
import {VaultManagerV2} from "../src/core/VaultManagerV2.sol";
import {DNft} from "../src/core/DNft.sol";
import {Dyad} from "../src/core/Dyad.sol";
import {Licenser} from "../src/core/Licenser.sol";
import {KerosineManager} from "../src/core/KerosineManager.sol";
import {UnboundedKerosineVault} from "../src/core/Vault.kerosine.unbounded.sol";

contract VaultManagerV2Test is VaultManagerTestHelper {
address private USER = makeAddr("USER");

Kerosine private keroseneToken = new Kerosine();
Licenser private licenser = new Licenser();
DNft dnft = new DNft();
Dyad dyadToken = new Dyad(licenser);
VaultManagerV2 vaultManagerV2 =
    new VaultManagerV2(dnft, dyadToken, licenser);
KerosineManager private kerosineManager = new KerosineManager();
UnboundedKerosineVault unboundedKeroseneVault =
    new UnboundedKerosineVault(
        vaultManagerV2,
        keroseneToken,
        dyadToken,
        kerosineManager
    );

function testUsersWontBeAbleToWithdrawTheirKerosene() external {
    vaultManagerV2.setKeroseneManager(kerosineManager);
    assertTrue(address(vaultManagerV2.keroseneManager()) != address(0));

    kerosineManager.add(address(wethVault));

    kerosineManager.add(address(unboundedKeroseneVault));

    vm.deal(USER, 100 ether);
    keroseneToken.transfer(USER, 100 ether);

    vm.startPrank(USER);
    uint userNftId = dnft.mintNft{value: 1 ether}(address(USER));

    vaultManagerV2.addKerosene(userNftId, address(unboundedKeroseneVault));

    console.log(
        "USER kerosene value before deposit: ",
        unboundedKeroseneVault.id2asset(userNftId)
    );

    keroseneToken.approve(address(vaultManagerV2), 10 ether);
    vaultManagerV2.deposit(
        userNftId,
        address(unboundedKeroseneVault),
        10 ether
    );

    console.log(
        "USER kerosene value after deposit: ",
        unboundedKeroseneVault.id2asset(userNftId)
    );

    console.log(
        "The user will try to withdraw their Kerosene, but this will revert due to unbounded Kerosene vault does not have an oracle"
    );
    vm.roll(10);
    vaultManagerV2.withdraw(
        userNftId,
        address(unboundedKeroseneVault),
        5 ether,
        USER
    );

    vm.stopPrank();
}
}
```

3. Run the test by using `forge test --match-contract VaultManagerV2Test -vvv`
4. As seen from the last part of the logs, the test fails due to the absence of an oracle in the Kerosene vaults.

```
├─ [9229] VaultManagerV2::withdraw(0, UnboundedKerosineVault: [0x1d1499e622D69689cdf9004d05Ec547d650Ff211], 5000000000000000000 [5e18], USER: [0xF921F4FA82620d8D2589971798c51aeD0C02c81a])
│   ├─ [557] DNft::ownerOf(0) [staticcall]
│   │   └─ ← [Return] USER: [0xF921F4FA82620d8D2589971798c51aeD0C02c81a]
│   ├─ [2623] Dyad::mintedDyad(VaultManagerV2: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 0) [staticcall]
│   │   └─ ← [Return] 0
│   ├─ [261] UnboundedKerosineVault::asset() [staticcall]
│   │   └─ ← [Return] Kerosine: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f]
│   ├─ [249] Kerosine::decimals() [staticcall]
│   │   └─ ← [Return] 18
│   ├─ [214] UnboundedKerosineVault::oracle() [staticcall]
│   │   └─ ← [Revert] EvmError: Revert
│   └─ ← [Revert] EvmError: Revert
└─ ← [Revert] EvmError: Revert
```

## Tools Used

VSCode, Foundry

## Recommended Mitigation Steps

Implement special function for withdrawing Kerosene tokens. 

## <a id='H-02'></a>H-02 Kerosene USD Value Calculation Flaw Causes Misrepresentation of Kerosene Value and Collateral Ratios

   # Lines of code

https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L269-L286


# Vulnerability details

## Impact

The Kerosene token price is deterministic, calculated as follows: `K_value = (TVL - DYAD_totalSupply) / K_supply`, meaning that Kerosene is as valuable as the degree of DYAD’s overcollateralization. However, in the current implementation of its USD value calculation in `VaultManagerV2`, there is a critical issue that will make the Kerosene price much higher than expected, thus leading to the following scenarios becoming possible:

- Malicious actors will be able to mint more than expected DYAD tokens.
- Malicious actors will be able to withdraw all of their exogenous collateral, and their collateral ratio will still be greater than `MIN_COLLATERIZATION_RATIO`.
- Liquidations of users with 0 exogenous collateral will be impossible due to their collateral ratio being more than `MIN_COLLATERIZATION_RATIO`.

## Proof of Concept

The issue arises from the fact that in the `VaultManagerV2::getKeroseneValue()`, we use the WETH vault implementation to `getUsdValue()` of Kerosene, which will basically return the WETH USD value instead, making the Kerosene price much higher than it should be. Therefore, the `VaultManagerV2::collatRatio()` will also be incorrectly calculated as it depends on `VaultManagerV2::getTotalUsdValue()` which depends on `getKeroseneValue()`, making the user's collateral ratio much higher than it should be, potentially resulting in significant consequences, as we will illustrate shortly. But first, let's understand how this issue comes about:

[VaultManagerV2.sol#L269-L286](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L269-L286)

```solidity
function getKeroseneValue(
  uint id
)
  public
  view
  returns (uint) {
    ...
    for (uint i = 0; i < numberOfVaults; i++) {
      Vault vault = Vault(vaultsKerosene[id].at(i)); // @audit we wrap to the wrong vault impl. The price will be the WETH price instead of Kerosene
      uint usdValue;
      if (keroseneManager.isLicensed(address(vault))) {
        usdValue = vault.getUsdValue(id);
      }
      totalUsdValue += usdValue;
    ...
}
}
```

NOTE that we wrap the kerosene vault address to a `Vault.sol` implementation instead to `Vault.kerosine.sol`. `Vault.sol` corresponds to the WETH vault in the system, see [here](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/script/deploy/Deploy.V2.s.sol#L49-L53). And therefore calling `Vault::getUsdValue()` will just return the USD value of the WETH token based on its [assetPrice()](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.sol#L91-L103) and the amount deposited by the particular user. Instead, we should get the USD value from the `Vault.kerosine.sol`, which will correctly return it as it calculates it based on the [deterministic Kerosene assetPrice()](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.kerosine.unbounded.sol#L50-L68).

### Potential consequences because of that incorrect price calculation

1. A malicious user will be able to withdraw all of their exogenous collateral and still have a collateral ratio >= `MIN_COLLATERIZATION_RATIO` as the last if statement will be true due to the inflated price of Kerosene.

```solidity
function withdraw(
  uint    id,
  address vault,
  uint    amount,
  address to
)
  public
    isDNftOwner(id)
{
  ...
  if (getNonKeroseneValue(id) - value < dyadMinted) revert NotEnoughExoCollat();
  _vault.withdraw(id, to, amount);
  if (collatRatio(id) < MIN_COLLATERIZATION_RATIO)  revert CrTooLow();
}
```

2. Liquidations of users with 0 exogenous collateral will be impossible as the if statement will be true due to the inflated price of Kerosene.

```solidity
function liquidate(
  uint id,
  uint to
)
  external
    isValidDNft(id)
    isValidDNft(to)
  {
    uint cr = collatRatio(id);
    if (cr >= MIN_COLLATERIZATION_RATIO) revert CrTooHigh();
    ...
}
```

## Tools Used

VSCode

## Recommended Mitigation Steps

Use `Vault.kerosine::getUsdValue()` instead of `Vault::getUsdValue()` to get the correct USD value of the Kerosene token.


## <a id='M-01'></a>M-01 Kerosene USD Value Calculation Flaw Causes Misrepresentation of Kerosene Value and Collateral Ratios

# Lines of code

https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.kerosine.unbounded.sol#L50-L68


# Vulnerability details

## Impact

The current implementation contains a flaw that causes the Kerosene `assetPrice()` function to malfunction. This flaw cascades into the breakdown of functions dependent on the calculation of collateral ratios.

## Proof of Concept

In the current implementation, a Kerosene vault address must be licensed by `KeroseneManager` before users can deposit their Kerosene tokens into it:

[VaultManagerV2.sol#L80-L91](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L80-L91)

```solidity
function addKerosene(
    uint    id,
    address vault
)
  external
    isDNftOwner(id)
{
  ...
  if (!keroseneManager.isLicensed(vault))                 revert VaultNotLicensed();
  ...
}
```

To license a vault, the `add()` function of `KerosineManager` must be called. This adds the specific vault to its EnumerableSet of `vaults`, effectively licensing the vault.

[KerosineManager.sol#L18-L26](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/KerosineManager.sol#L18-L26)

```solidity
function add(
  address vault
)
  external
    onlyOwner
{
  if (vaults.length() >= MAX_VAULTS) revert TooManyVaults();
  if (!vaults.add(vault))            revert VaultAlreadyAdded();
}
```

But this will lead to an issue when we attempt to calculate the Kerosene price. Let's examine how this occurs. The Kerosene price is determined based on DYAD's overcollateralization, making it a deterministic value. In the calculation, only exogenous collateral vaults should be included, while Kerosene vaults should be excluded. However, due to the previous addition of Kerosene vaults to the [KeroseneManager's set of vaults](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/KerosineManager.sol#L12) for licensing them, they are inadvertently included in the calculation. And, when we call `address[] memory vaults = kerosineManager.getVaults();`, it returns these vaults as well.

[Vault.kerosine.unbounded.sol#L50-L68](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.kerosine.unbounded.sol#L50-L68)

```solidity
function assetPrice()
  public
  view
  override
  returns (uint) {
    uint tvl;
    address[] memory vaults = kerosineManager.getVaults(); // @audit this will get the licensed Kerosene vaults as well
    uint numberOfVaults = vaults.length;
    for (uint i = 0; i < numberOfVaults; i++) {
      Vault vault = Vault(vaults[i]);
      tvl += vault.asset().balanceOf(address(vault))
              * vault.assetPrice() * 1e18
              / (10**vault.asset().decimals())
              / (10**vault.oracle().decimals());
    }
    uint numerator   = tvl - dyad.totalSupply();
    uint denominator = kerosineDenominator.denominator();
    return numerator * 1e8 / denominator;
}
```

However, this won't result in incorrect calculations. Instead, the function will revert because Kerosene vaults lack an oracle. When attempting to invoke `10**vault.oracle().decimals()`, the function will simply revert. Consequently, all functions dependent on the Kerosene asset price calculation will also fail.

Below is a PoC test demonstrating that users are unable to mint DYAD tokens due to this issue, as mint requires collateral ratio to be calculated. Please follow the bellow steps to reproduce it locally.

1. In the test folder create a new file called `KerosenePriceIssueTest.t.sol`
2. Copy the PoC content and paste it there
3. Run the test by using `forge test --match-test testUsersWontBeAbleToMint -vvvv`

```solidity
pragma solidity =0.8.17;

import {VaultManagerTestHelper} from "./VaultManagerHelper.t.sol";
import {Test, console} from "forge-std/Test.sol";
import {Kerosine} from "../src/staking/Kerosine.sol";
import {VaultManagerV2} from "../src/core/VaultManagerV2.sol";
import {DNft} from "../src/core/DNft.sol";
import {Dyad} from "../src/core/Dyad.sol";
import {Licenser} from "../src/core/Licenser.sol";
import {KerosineManager} from "../src/core/KerosineManager.sol";
import {UnboundedKerosineVault} from "../src/core/Vault.kerosine.unbounded.sol";
import {Vault} from "../src/core/Vault.sol";
import {IAggregatorV3} from "../src/interfaces/IAggregatorV3.sol";
import {OracleMock} from "./OracleMock.sol";

contract KerosenePriceIssueTest is VaultManagerTestHelper {
  address private USER = makeAddr("USER");

  Kerosine private keroseneToken = new Kerosine();
  Licenser private licenser = new Licenser();
  DNft dnft = new DNft();
  Dyad dyadToken = new Dyad(licenser);
  VaultManagerV2 vaultManagerV2 =
      new VaultManagerV2(dnft, dyadToken, licenser);
  KerosineManager private kerosineManager = new KerosineManager();
  UnboundedKerosineVault unboundedKeroseneVault =
      new UnboundedKerosineVault(
          vaultManagerV2,
          keroseneToken,
          dyadToken,
          kerosineManager
      );

  function testUsersWontBeAbleToMint() external {
      OracleMock oracleMock = new OracleMock(1000e8);
      Vault myWethVault = new Vault(
          vaultManagerV2,
          weth,
          IAggregatorV3(address(oracleMock))
      );

      vaultManagerV2.setKeroseneManager(kerosineManager);
      assertTrue(address(vaultManagerV2.keroseneManager()) != address(0));

      licenser.add(address(myWethVault));
      kerosineManager.add(address(myWethVault));
      licenser.add(address(vaultManagerV2));

      kerosineManager.add(address(unboundedKeroseneVault));

      vm.deal(USER, 100 ether);
      weth.mint(USER, 100 ether);
      keroseneToken.transfer(USER, 100 ether);

      vm.startPrank(USER);
      uint userNftId = dnft.mintNft{value: 1 ether}(address(USER));

      vaultManagerV2.add(userNftId, address(myWethVault));
      weth.approve(address(vaultManagerV2), 50 ether);
      vaultManagerV2.deposit(userNftId, address(myWethVault), 50 ether);

      console.log("Kerosene vault address", address(unboundedKeroseneVault));
      console.log("WETH vault address", address(myWethVault));
      vaultManagerV2.addKerosene(userNftId, address(unboundedKeroseneVault));
      keroseneToken.approve(address(vaultManagerV2), 20 ether);
      vaultManagerV2.deposit(
          userNftId,
          address(unboundedKeroseneVault),
          20 ether
      );

      vm.roll(10);
      vaultManagerV2.mintDyad(userNftId, 5 ether, USER);
      vm.stopPrank();
  }
}
```

4. NOTE that when we call `KeroseneManager::getVaults()`, it returns two addresses instead of just the WETH vault address as expected. It also returns the Kerosene vault address.

```
[FAIL. Reason: EvmError: Revert] testUsersWontBeAbleToMint() (gas: 1590764)
Logs:
Kerosene vault address 0x1d1499e622D69689cdf9004d05Ec547d650Ff211
WETH vault address 0x96d3F6c20EEd2697647F543fE6C08bC2Fbf39758
...
KerosineManager::getVaults() [staticcall]
│   │   │   └─ ← [Return] [0x96d3F6c20EEd2697647F543fE6C08bC2Fbf39758, 0x1d1499e622D69689cdf9004d05Ec547d650Ff211]
```

5. The when we try to call the oracle on the Kerosene vault, the function reverts:

```

  │   ├─ [10155] UnboundedKerosineVault::getUsdValue(0) [staticcall]
  │   │   ├─ [1342] KerosineManager::getVaults() [staticcall]
  │   │   │   └─ ← [Return] [0x96d3F6c20EEd2697647F543fE6C08bC2Fbf39758, 0x1d1499e622D69689cdf9004d05Ec547d650Ff211]
  │   │   ├─ [249] Vault::oracle() [staticcall]
  │   │   │   └─ ← [Return] OracleMock: [0xD16d567549A2a2a2005aEACf7fB193851603dd70]
  │   │   ├─ [144] OracleMock::decimals() [staticcall]
  │   │   │   └─ ← [Return] 8
  │   │   ├─ [250] Vault::asset() [staticcall]
  │   │   │   └─ ← [Return] ERC20Mock: [0x03A6a84cD762D9707A21605b548aaaB891562aAb]
  │   │   ├─ [271] ERC20Mock::decimals() [staticcall]
  │   │   │   └─ ← [Return] 18
  │   │   ├─ [1431] Vault::assetPrice() [staticcall]
  │   │   │   ├─ [420] OracleMock::latestRoundData() [staticcall]
  │   │   │   │   └─ ← [Return] 1, 100000000000 [1e11], 1, 1, 1
  │   │   │   └─ ← [Return] 100000000000 [1e11]
  │   │   ├─ [250] Vault::asset() [staticcall]
  │   │   │   └─ ← [Return] ERC20Mock: [0x03A6a84cD762D9707A21605b548aaaB891562aAb]
  │   │   ├─ [552] ERC20Mock::balanceOf(Vault: [0x96d3F6c20EEd2697647F543fE6C08bC2Fbf39758]) [staticcall]
  │   │   │   └─ ← [Return] 50000000000000000000 [5e19]
  │   │   ├─ [214] UnboundedKerosineVault::oracle() [staticcall]
  │   │   │   └─ ← [Revert] EvmError: Revert
  │   │   └─ ← [Revert] EvmError: Revert
  │   └─ ← [Revert] EvmError: Revert
  └─ ← [Revert] EvmError: Revert

```

## Tools Used

VSCode, Foundry

## Recommended Mitigation Steps

Add a `mapping(address=>bool)` and new functions for licensing - `license(address vault)` and `unlicense(address vault)` in `KeroseneManager.sol`, and when licensing vaults, call the `license()` function and simply set the corresponding bool value to true. Use `KeroseneManager::add()` only for vaults that should be included in the Kerosene asset price calculations, not for licensing purposes.

## <a id='M-02'></a>M-02 Kerosene Price Manipulation Vulnerability Leading to Protocol Instability

# Lines of code

https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/Vault.kerosine.unbounded.sol#L50-L68


# Vulnerability details

## Impact

Manipulating the price of the Kerosene token could create unfair arbitrage opportunities for malicious actors with sufficient financial resources. This manipulation could adversely affect users' collateral ratios, potentially leading to their liquidation by malicious actors seeking to steal their collateral. This price manipulation attack vector could severely damage the overall health, functionality, and reputation of the system.

## Proof of Concept

The protocol sets the price of Kerosene deterministically based on the amount of exogenous collateral deposited within it. This price is determined by the following formula:

$$ X = \frac{C - D}{K} $$

- **X:** the value of a single Kerosene token
- **C:** the total USD value of all exogenous collateral in the protocol (TVL).
- **D:** the total supply of DYAD stablecoins
- **K:** the total supply of Kerosene tokens

---

However, this approach hides a significant risk: Kerosene's value is vulnerable to manipulation. Well-funded malicious actor could exploit this vulnerability, to take an advantage of unfair arbitrage opportunities to profit while harming other users and the protocol. Let's see an example step by step exploit scenario:

1. The malicious actor purchases a significant quantity of Kerosene tokens and deposits them into the protocol.
2. The malicious actor deposits significant collateral and launches a deceptive campaign, showcasing the rising price of Kerosene to lure more users into the DYAD system and potentially liquidate their positions later.
3. Other users are lured in by the campaign and begin depositing Kerosene and other exogenous collateral to mint DYAD tokens.
4. Once the malicious actor deems that there is sufficient incentive, they withdraw their Kerosene tokens, which have now increased in price, and also withdraw all of their exogenous collateral. Since they did not mint any DYAD tokens, they are able to do so without consequence, causing a significant drop in the price of Kerosene.

(refer to the withdraw function below to confirm that this type of withdrawal is indeed possible.)

[VaultManagerV2.sol#L134-L153](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L134-L153)

```solidity
 function withdraw(
     uint id,
     address vault,
     uint amount,
     address to
 ) public isDNftOwner(id) {
    ...
     _vault.withdraw(id, to, amount);
     if (collatRatio(id) < MIN_COLLATERIZATION_RATIO) revert CrTooLow();
 }

```

[VaultManagerV2.sol#L230-L239](https://github.com/code-423n4/2024-04-dyad/blob/cd48c684a58158de444b24854ffd8f07d046c31b/src/core/VaultManagerV2.sol#L230-L239)

```solidity
function collatRatio(uint id) public view returns (uint) {
     uint _dyad = dyad.mintedDyad(address(this), id);
     if (_dyad == 0) return type(uint).max;
     return getTotalUsdValue(id).divWadDown(_dyad);
 }
```

5. Subsequently, the actor can liquidate users who have become vulnerable to liquidation due to the manipulated price, thereby seizing their collateral.
6. This allows the malicious actor to sell Kerosene tokens at inflated prices on secondary markets, earning profits, while also acquiring additional exogenous collateral through liquidations.
7. However, these actions severely undermine the stability of the protocol. Users witnessing the drastic price drop may panic and sell their Kerosene tokens, potentially leading to insolvency within the protocol.

## Tools Used

VSCode

## Recommended Mitigation Steps

Mitigating this vulnerability is not straightforward. While implementing mechanisms to track suspicious transactions could be considered, the protocol team should also explore more stable methods for determining the Kerosene price, as the current implementation is vulnerable to this type of attack.
