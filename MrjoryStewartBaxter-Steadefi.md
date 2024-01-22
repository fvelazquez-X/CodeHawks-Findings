# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Missing checks for deposits/withdrawals above the Max PnL factor and Max PnL factor not defined](#M-01)
- ## Low Risk Findings
    - ### [L-01. Unhandled DoS when access to Chainlik oracle is blocked](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 1
   - Low: 1



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing checks for deposits/withdrawals above the Max PnL factor and Max PnL factor not defined            



## Summary

Following the deposit/withdraw flows with the before/after checkings, there are currently no verifications that ensure deposits/withdrawals would no go above the Max PnL factor. 

## Vulnerability Details

In this two-step transaction process (where GMX is based for various asset transfer transactions), there are multiple junctures where potential reversals could occur, necessitating the Vault's ability to manage them appropriately.

These can be categorized at a glance in before and after checkings and are mostly contained in `GMXChecks.sol`:

For example, consider this `beforeDeposit` check that verifies if the user did not send enough execution fee:

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXChecks.sol#L62-L63

```
 function beforeDepositChecks(
    GMXTypes.Store storage self,
    uint256 depositValue
  ) external view {
    
    if (self.depositCache.depositParams.executionFee < self.minExecutionFee)
      revert Errors.InsufficientExecutionFeeAmount();
```

The issue arises precisely when executing deposits or withdrawals in this two-step flow and going for their respective checks under `beforeDepositChecks` and `beforeWithdrawChecks`, there are currently no verifications in place that a deposit should not go above the Max PnL factor for deposits, and at the same time there is no check available for verifying that a withdrawal should not go above Max PnL factor for withdrawals.

Nevertheless, as this flag/key of Max PnL factor is currently not defined in the codebase, and the verifications missing require this factor/cap to be established in the first place, a suggestion on how to calculate it can be found in https://blog.whitebit.com/en/what-is-profit-and-loss/. Also a good reference (which is referring to GMX) is https://www.collider.vc/post/gmx-granted-million-dollar-bug-bounty-to-collider-the-bug-aftermath 

## Impact

As the market token price is influenced by both the total value of assets within the pool and the aggregate pending profit and loss of traders' active positions, it is necesary to establish the necessary caps used to calculate the market token price depending if it is a deposit or a withdrawal. 

## Tools Used

Manual Review 

## Recommendations

Establish the proper PnL factor caps when calculating the market token price for deposits/withdrawals and ensure the verifications are in place under `beforeDepositChecks` and `beforeWithdrawChecks` in `GMXChecks.sol`

# Low Risk Findings

## <a id='L-01'></a>L-01. Unhandled DoS when access to Chainlik oracle is blocked            



## Summary

In certain exceptional scenarios, oracles may become temporarily unavailable. As a result, invoking the `latestRoundData` function could potentially revert without a proper error handling.

## Vulnerability Details

Steadefi documentation gives special focus on Chainlink price feed dependency, (https://github.com/Cyfrin/2023-10-SteadeFi/tree/main "Additional Context"). The concern stems from the potential for Chainlink multisignature entities to deliberately block the access to the price feed. In such a situation, using the `latestRoundData` function could lead to an unexpected revert.

In certain extraordinary situations, Chainlink has already proactively suspended particular oracles. To illustrate, in the case of the UST collapse incident, Chainlink chose to temporarily halt the UST/ETH price oracle to prevent the propagation of incorrect data to various protocols.

Additionally, this danger has been highlighted and very well documented by OpenZeppelin in https://blog.openzeppelin.com/secure-smart-contract-guidelines-the-dangers-of-price-oracles. For our current scenario:

*"While currently there’s no whitelisting mechanism to allow or disallow contracts from reading prices, powerful multisigs can tighten these access controls. In other words, the multisigs can immediately block access to price feeds at will. Therefore, to prevent denial of service scenarios, it is recommended to query ChainLink price feeds using a defensive approach with Solidity’s try/catch structure. In this way, if the call to the price feed fails, the caller contract is still in control and can handle any errors safely and explicitly".*

As a result and taking into consideration the recommendation from OpenZepplin, it is essential to thoroughly tackle this matter within the codebase, as it directly relates to many functionalities of the system which are based on the oracle's output.

Another example to check this vulnerability can be consulted in https://solodit.xyz/issues/m-18-protocols-usability-becomes-very-limited-when-access-to-chainlink-oracle-data-feed-is-blocked-code4rena-inverse-finance-inverse-finance-contest-git 

## Proof of Concept

As previously discussed, to mitigate the potential risks related to a denial-of-service situation, it is recommended to implement a try-catch mechanism when querying Chainlink prices in the `_getChainlinkResponse` function within `ChainlinkARBOracle.sol` (link to code below). By adopting this approach, in case there's a failure in invoking the price feed, the caller contract retains control and can effectively handle any errors securely and explicitly.

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/ChainlinkARBOracle.sol#L188-L194

```

 (
      uint80 _latestRoundId,
      int256 _latestAnswer,
      /* uint256 _startedAt */,
      uint256 _latestTimestamp,
      /* uint80 _answeredInRound */
    ) = AggregatorV3Interface(_feed).latestRoundData();

```

## Impact

In the event of a malfunction or cessation of operation of a configured Oracle feed, attempting to check for the `latestRoundData` will result in a revert that must be managed manually by the system.

## Tools Used

Manual review

## Recommendations

Wrap the invocation of the `latestRoundData()` function within a `try-catch` structure rather than directly calling it. In situations where the function call triggers a revert, the catch block can be utilized to trigger an alternative oracle or handle the error in a manner that aligns with the system's requirements.


