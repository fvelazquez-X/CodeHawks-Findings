# DittoETH - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)


- ## Low Risk Findings
    - ### [L-01. Unhandled chainlink revert in case its multisigs block access to price feeds](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 0
   - Low: 1



		


# Low Risk Findings

## <a id='L-01'></a>L-01. Unhandled chainlink revert in case its multisigs block access to price feeds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOracle.sol#L20-L62

## Summary

In some extreme cases, oracles can be taken offline or token prices can fall to zero. Therefore a call to `latestRoundData` could potentially revert and none of the circuit breakers would fallback to query any prices automatically.

## Vulnerability Details

According to Ditto’s documentation in https://dittoeth.com/technical/oracles, there are two circuit breaking events if Chainlink data becomes unusable: Invalid Fetch Data and Price Deviation. 

The issue arises from the possibility that Chainlink multisignature entities might intentionally block access to the price feed. In such a scenario, the invocation of the `latestRoundData` function could potentially trigger a revert, rendering the circuit-breaking events ineffective in mitigating the consequences, as they would be incapable of querying any price data or specific information.

In certain exceptional circumstances, Chainlink has already taken the initiative to temporarily suspend specific oracles. As an illustrative instance, during the UST collapse incident, Chainlink opted to halt the UST/ETH price oracle to prevent the dissemination of erroneous data to various protocols.

Additionally, these dangerous oracle's scenarios are very well documented by OpenZeppelin in https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles. For our context:

*"While currently there’s no whitelisting mechanism to allow or disallow contracts from reading prices, powerful  multisigs can tighten these access controls. In other words, the multisigs can immediately block access to price feeds at will. Therefore, to prevent denial of service scenarios, it is recommended to query ChainLink price feeds using a defensive approach with Solidity’s try/catch structure. In this way, if the call to the price feed fails, the caller contract is still in control and can handle any errors safely and explicitly".*

Although a fallback mechanism, specifically the TWAP, is in place to uphold system functionality in the event of Chainlink failure, it is imperative to note that Ditto's documentation **explicitly underscores its substantial reliance on oracles**. Consequently, it is imperative to address this issue comprehensively within the codebase, given that it pertains to one of the fundamental functionalities of the environment. 

## Proof of Concept

As mentioned above, In order to mitigate the potential risks associated with a denial-of-service scenario, it is advisable to employ a `try-catch` mechanism when querying Chainlink prices in the function `getOraclePrice` under LibOracle.sol. Through this approach, in the event of a failure in the invocation of the price feed, the caller contract retains command and can adeptly manage any errors in a secure and explicit manner.

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOracle.sol#L25-L32

```
 (
            uint80 baseRoundID,
            int256 basePrice,
            /*uint256 baseStartedAt*/
            ,
            uint256 baseTimeStamp,
            /*uint80 baseAnsweredInRound*/
        ) = baseOracle.latestRoundData();
```

Here I enumerate some of the core functions that will be affected in case of an unhandled oracle revert:

Function createMarket under OwnerFacet.sol: 

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/OwnerFacet.sol#L47-L68

Function updateOracleAndStartingShort under LibOrders.sol:

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOrders.sol#L812-L816 

Function getShortIdAtOracle under ViewFaucet.sol:

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/ViewFacet.sol#L173-L187 


## Impact

If a configured Oracle feed has malfunctioned or ceased operating, it will produce a revert when checking for `latestRoundData` that would need to be manually handled by the system.


## Tools Used

Manual Review

## Recommendations

Encase the invocation of the function `latestRoundData()` within a `try-catch` construct instead of invoking it directly. In circumstances where the function call results in a revert, the catch block may serve the purpose of invoking an alternative oracle or managing the error in a manner that is deemed appropriate for the system. 






