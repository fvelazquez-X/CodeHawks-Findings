# Foundry DeFi Stablecoin CodeHawks Audit Contest - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. staleCheckLatestRoundData() does not check the status of the Arbitrum sequencer in Chainlink feeds.](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Cyfrin

### Dates: Jul 24th, 2023 - Aug 5th, 2023

[See more contest details here](https://www.codehawks.com/contests/cljx3b9390009liqwuedkn0m0)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 1
   - Low: 0



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. staleCheckLatestRoundData() does not check the status of the Arbitrum sequencer in Chainlink feeds.            



## Summary

Given that the contract will be deployed on any EVM chain, when utilizing Chainlink in L2 chains like Arbitrum, it's important to ensure that the prices provided are not falsely perceived as fresh particularly in scenarios where the sequencer might be non-operational. Hence, a critical step involves confirming the active status of the sequencer before trusting the data returned by the oracle.


## Vulnerability Details

In the event of an Arbitrum Sequencer outage, the oracle data may become outdated, potentially leading to staleness. While the function  staleCheckLatestRoundData() provides checks if a price is stale, it does not check if Arbirtrum Sequencer is active. Since OracleLib.sol library is used to check the Chainlink Oracle for stale data, it is important to add this verification. You can review Chainlink docs on L2 Sequencer Uptime Feeds for more details on this. https://docs.chain.link/data-feeds/l2-sequencer-feeds 


## Impact

In the scenario where the Arbitrum sequencer experiences an outage, the protocol will enable users to maintain their operations based on the previous (stale) rates.


## Tools Used

Manual Review

## Recommendations


There is a code example on Chainlink docs for this scenario: https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code. 
For illustrative purposes this can be:

```
function isSequencerAlive() internal view returns (bool) {
    (, int256 answer, uint256 startedAt,,) = sequencer.latestRoundData();
    if (block.timestamp - startedAt <= GRACE_PERIOD_TIME || answer == 1)
        return false;
    return true;
}


function staleCheckLatestRoundData(AggregatorV3Interface priceFeed)
        public
        view
        returns (uint80, int256, uint256, uint256, uint80)
    {
require(isSequencerAlive(), "Sequencer is down");
       ....//remaining parts of the function
```





