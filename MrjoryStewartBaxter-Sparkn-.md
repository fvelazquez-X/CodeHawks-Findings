# Sparkn  - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Organizer (possible sponsor) can recover part of its funds after setting and funding the contest.](#M-01)
    - ### [M-02. Commission fees and remaining tokens can get stucked in contract if stadium address is blacklisted.](#M-02)
- ## Low Risk Findings
    - ### [L-01. Funds can be lost in case winners == address(0).](#L-01)
    - ### [L-02. Tokens utilizing a Blocklist, such as USDC/USDT, can prevent fund being received by winners, or sent by sponsors.](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: CodeFox Inc.

### Dates: Aug 21st, 2023 - Aug 29th, 2023

[See more contest details here](https://www.codehawks.com/contests/cllcnja1h0001lc08z7w0orxx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 2
   - Low: 2



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Organizer (possible sponsor) can recover part of its funds after setting and funding the contest.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L125

## Summary

It is possible for the organizer to obtain/get funds back within the distribute function itself and by calling either deploy proxy and distribute or use meta transaction to send the signature to someone else to deploy proxy and distribute, as there is no check that impedes the organizer to be included into the winners array. 

## Vulnerability Details

According to “More Context” in README of Sparkn repo “If a contest is created and funded, there is no way to refund”. This categorization includes any percentage of the funds. But consider the following scenario:

1.- A new contest is set by the owner using setContest() function specifying all the pertinent parameters. 
2.- The contest is funded by an organizer sending the corresponding ERC20 tokens.
3.- At this point, there are two possibilities for the sponsor (aka organizer) to get its funds back:

*Since there is no check that ensures organizers cannot be included into the winners array (the only existing verification within this array is related to its length):*

1.- If the organizer is included in the winners array, and its length equals one (meaning there will be only the organizer in the array), he can call the deployProxyandDistribute function, or even deployProxyAndDistributeBySignature to distribute the prize only to him (minus COMISSION FEE). It might be possible even for the owner to execute this transaction using the distributeByOwner, but for the sake of the scenario we can focus only with the previous functions. 

2.- If the organizer is included in the winners array, and its length is greater than one (meaning there are more winners) the distribution will include of course other addresses as well, but the organizer might be able to still obtain a percentage of his funds back. 

Nevertheless since the organizer is able to call the deploy functions, he can get this funds back after the contest close time. 

The following PoC under test *testIfAllConditionsMetThenUsdcSendingCallShouldSuceed* proofs that a sponsor can be added to the winners array without restriction getting a percentage of his funds back. Minor modifications to the original test file where made to include a sponsor into the winners array and after running the complete evlauation where all the conditions are met, at the end of the test the balance of the sponsor will be increased because it is considered a winner:


```
function testIfAllConditionsMetThenUsdcSendingCallShouldSuceed()
        public
        setUpContestForNameAndSentAmountToken("James", jpycv2Address, 10000 ether)
    {
        // before
        assertEq(MockERC20(usdcAddress).balanceOf(address(user1)), 0);
        assertEq(MockERC20(usdcAddress).balanceOf(address(sponsor)), 0);

        // deploy proxy
        bytes32 randomId_ = keccak256(abi.encode("James", "001"));
        bytes memory data = createDataToDistributeJpycv2();
        vm.startPrank(organizer);
        deployedProxy = proxyFactory.deployProxyAndDistribute(randomId_, address(distributor), data);
        vm.stopPrank();

        proxyWithDistributorLogic = Distributor(address(deployedProxy));

        // prepare data
        address[] memory winners = new address[](2);
        winners[0] = user1;
        // Sponsor is included in the winner's array
        winners[1] = sponsor; 
        uint256[] memory percentages_ = new uint256[](2);
        percentages_[0] = 9000;
        percentages_[1] = 500;

        // sponsor send token to proxy by mistake
        vm.startPrank(sponsor);
        MockERC20(usdcAddress).transfer(deployedProxy, 1000 ether);
        vm.stopPrank();

        // If all conditions met then call should succeed
        vm.startPrank(address(proxyFactory));
        proxyWithDistributorLogic.distribute(usdcAddress, winners, percentages_, "");
        vm.stopPrank();

        // after this, token should be distributed correctly as expected
        assertEq(MockERC20(usdcAddress).balanceOf(address(user1)), 900 ether);
        assertEq(MockERC20(usdcAddress).balanceOf(address(sponsor)), 50 ether);
        assertEq(MockERC20(usdcAddress).balanceOf(stadiumAddress), 50 ether);
    }

```

At the beginning of the test the balance of user1 is 0 and we can see sponsor has already some balance:

```
[2563] MockERC20::balanceOf(user1: [0x000000000000000000000000000000000000000E]) [staticcall]
    │   └─ ← 0
    ├─ [2563] MockERC20::balanceOf(sponsor: [0x000000000000000000000000000000000000000C]) [staticcall]
    │   └─ ← 10000000000000000000000 [1e22]

```

But at the end, when contest is finished and prizes are distributed to winners, since sponsor is included into the array, and this is allowed by the contest, his balance is increased:

```
─ [563] MockERC20::balanceOf(user1: [0x000000000000000000000000000000000000000E]) [staticcall]
    │   └─ ← 900000000000000000000 [9e20]
    ├─ [563] MockERC20::balanceOf(sponsor: [0x000000000000000000000000000000000000000C]) [staticcall]
    │   └─ ← 9050000000000000000000 [9.05e21]
```


## Impact

Considering that the organizer can be put into the winners array, it is possible for him to violate the philosophy of “supporters first”, meaning there will be a way to recover the majority (or part) of his funds after a contest is created and funded and after the contest close time is reached. 

## Tools Used

Static review

## Recommendations

Ensure that winners != organizers during the creation of the contest. 
## <a id='M-02'></a>M-02. Commission fees and remaining tokens can get stucked in contract if stadium address is blacklisted.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L163-L165

## Summary

The Comission fee and remaining tokens can get stucked in the contract if stadium address gets blacklisted. Additonally, since the function _commisionTransfer is included inside the _distribute function, this will end with a revert because transfering this fee/remaining assets is the last step of the distribution process. 

## Vulnerability Details

The issue arises  when the stadium address is unaware of being included in a blocklist, before receiving the fee and/or remainings. Neither the Distributor.sol contract nor the ProxyFactory.sol currently incorporate proper verifications to mitigate this specific blocklist scenario. The consequence of this is that the primary task of _comissionTransfer (avoid dust remaining) will fail, and moreover since this function is also called inside the _distribute function, it will also produce a failure. 

The following PoC can illustrate the issue. For simulating the block, we will use the the HelperContract.t.sol by simply putting the stadium address to 0 declared in the test contract, (for simplifying the scenario). In this way using the `testIfAllConditionsMetThenUsdcSendingCallShouldSuceed`, we will see that while most of the contest process is correctly parsed, the last part corresponding to the distribution of the comission fee will fail as the stadium address is invalid. 

```
// users
    address public stadiumAddress = address(0);
    address public factoryAdmin = makeAddr("factoryAdmin");
    address public tokenMinter = makeAddr("tokenMinter");
    address public organizer = address(11);
```

```
  function testIfAllConditionsMetThenUsdcSendingCallShouldSuceed()
        public
        setUpContestForNameAndSentAmountToken("James", jpycv2Address, 10000 ether)
    {
        // Code chunk not displayed       
        //....

        // after this, token should be distributed correctly as expected
        assertEq(MockERC20(usdcAddress).balanceOf(address(user1)), 900 ether);
        assertEq(MockERC20(usdcAddress).balanceOf(address(user2)), 50 ether);
          (, , uint256 commissionFee, ) =
            proxyWithDistributorLogic.getConstants();
        assertEq(commissionFee, 500);
        assertEq(MockERC20(usdcAddress).balanceOf(stadiumAddress), 50 ether);
    }
```

*** The comission fee is stucked inside the contract as we can verify using the `getConstants()` method.


```
 ├─ [563] MockERC20::balanceOf(user1: [0x000000000000000000000000000000000000000E]) [staticcall]
    │   └─ ← 900000000000000000000 [9e20]
    ├─ [563] MockERC20::balanceOf(user2: [0x000000000000000000000000000000000000000F]) [staticcall]
    │   └─ ← 50000000000000000000 [5e19]
    ├─ [502] 0x9824278B4b41B8f00c96f05BA590Af1a5cE0E326::getConstants() [staticcall]
    │   ├─ [249] Distributor::getConstants() [delegatecall]
    │   │   └─ ← ProxyFactory: [0x5FbDB2315678afecb367f032d93F642f64180aa3], stadium: [0xFa6523E66a411fC3d12656507877B8cbf1E39aC0], 500, 1
    │   └─ ← ProxyFactory: [0x5FbDB2315678afecb367f032d93F642f64180aa3], stadium: [0xFa6523E66a411fC3d12656507877B8cbf1E39aC0], 500, 1
    ├─ [2563] MockERC20::balanceOf(0x0000000000000000000000000000000000000000) [staticcall]
    │   └─ ← 0
    ├─ emit log(: Error: a == b not satisfied [uint])
    ├─ emit log_named_uint(key:       Left, val: 0)
    ├─ emit log_named_uint(key:      Right, val: 50000000000000000000 [5e19])
    ├─ [0] VM::store(VM: [0x7109709ECfa91a80626fF3989D68f67F5b1DD12D], 0x6661696c65640000000000000000000000000000000000000000000000000000, 0x0000000000000000000000000000000000000000000000000000000000000001) 
    │   └─ ← ()
    └─ ← ()
```


## Impact

The comission fee and all remaining tokens can get stuck in the contract. Additionally the _distribute function will fail as the last step where comissions are transfered will not work. 

## Tools Used

Manual Review

## Recommendations

Implement the necessary checks for the possible blocklist of the stadium address that can prevent this failure. 

# Low Risk Findings

## <a id='L-01'></a>L-01. Funds can be lost in case winners == address(0).            



## Summary

If the winners’ addresses are erroneously set with the zero address, the distribution will result in a loss of funds. 

## Vulnerability Details

The contract can be successfully deployed with a zero address for the winners (aka supporters), which should not be allowed. 

In the contracts, there are specific checks for nonzero address in

the organizer and implementation address: https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L109 

the token address:
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L120 

the factory and stadium address:
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L77 

the proxy address:
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L212 

and the whitelisted tokens addresses:
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/ProxyFactory.sol#L84 

But even though there is a specific check that reverts when the winners array length is equal to 0 (https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L125), the winner’s addresses itself are not verified against the zero address. 

Consider the following PoC using the HelperContract.t.sol. As mentioned it is possible to make a deployment using the address(0) for the winners as nothing prevents to do it and there is no check implemented; therefore we can make a minor modification to change the variable `user1` to `address(0)`, where all the actors are declared to test our scenario:

```
 // users
    address public stadiumAddress = makeAddr("stadium");
    address public factoryAdmin = makeAddr("factoryAdmin");
    address public tokenMinter = makeAddr("tokenMinter");
    address public organizer = address(11);
    address public sponsor = address(12);
    address public supporter = address(13);

 // Change user1 to have address(0)
    address public user1 = address(0);
    address public user2 = address(15);
    address public user3 = address(16);
``` 

After executing the test where all conditions are met, so the contest should succeed `testIfAllConditionsMetThenUsdcSendingCallShouldSuceed`, we get a revert when distributing the prizes because one of the winners is having an address(0), which we know in advance is user1, as this user is used throughout the function test:

```
[187707] ProxyTest::testIfAllConditionsMetThenUsdcSendingCallShouldSuceed() 

///// output not displayed

[11147] 0x9824278B4b41B8f00c96f05BA590Af1a5cE0E326::distribute(MockERC20: [0x90193C961A926261B756D1E5bb255e67ff9498A1], [0x0000000000000000000000000000000000000000], [9500], 0x) 
    │   │   ├─ [8345] Distributor::distribute(MockERC20: [0x90193C961A926261B756D1E5bb255e67ff9498A1], [0x0000000000000000000000000000000000000000], [9500], 0x) [delegatecall]
    │   │   │   ├─ [2575] ProxyFactory::whitelistedTokens(MockERC20: [0x90193C961A926261B756D1E5bb255e67ff9498A1]) [staticcall]
    │   │   │   │   └─ ← true
    │   │   │   ├─ [563] MockERC20::balanceOf(0x9824278B4b41B8f00c96f05BA590Af1a5cE0E326) [staticcall]
    │   │   │   │   └─ ← 10000000000000000000000 [1e22]
    │   │   │   ├─ [597] MockERC20::transfer(user1: [0x0000000000000000000000000000000000000000], 9500000000000000000000 [9.5e21]) 
    │   │   │   │   └─ ← "ERC20: transfer to the zero address"
    │   │   │   └─ ← "ERC20: transfer to the zero address"
    │   │   └─ ← "ERC20: transfer to the zero address"
    │   └─ ← "ProxyFactory__DelegateCallFailed()"
    └─ ← "ProxyFactory__DelegateCallFailed()"
```
## Impact

Sponsor funds can get lost in contract if winners’ addresses are erroneously set as a zero address.

## Tools Used

Static review

## Recommendations

There should be zero address checkings in place for the winners (supporters) to prevent loss of funds.

## <a id='L-02'></a>L-02. Tokens utilizing a Blocklist, such as USDC/USDT, can prevent fund being received by winners, or sent by sponsors.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-08-sparkn/blob/main/src/Distributor.sol#L137-L139

## Summary

USDC/USDT token has implemented an administratively owned blocklist feature effectively restricting users from transmitting or acquiring funds within this Token. This is important to consider as it was clarified that ERC20 stable coins such as USDC and USDT will be used in SPARKN.

## Vulnerability Details

Note that https://github.com/d-xo/weird-erc20#tokens-with-blocklists shows that:

*Some tokens (e.g. USDC, USDT) have a contract level admin controlled address blocklist. If an address is blocked, then transfers to and from that address are forbidden.*

The vulnerability may arise when the supporters (winners in this case), or soponsors, are unaware of being included in the USDC/USDT blocklist, before receiving a prize, or when sending funds. Neither the Distributor.sol contract nor the ProxyFactory.sol currently incorporate proper verifications to mitigate this specific blocklist scenario.

## Impact

Due to the lack of verification for the supporters and sponsors presence on the blocklist, the contract might be unable to distribute prizes or receive funds.

## Tools Used

Static Review

## Recommendations

To address this vulnerability, it is advisable to implement a check, (possibly within the distribute function of the Distributor.sol contract), specifically verifying whether the Token used has a blocklist and consequently checking that receiver/sender is not blacklisted.


