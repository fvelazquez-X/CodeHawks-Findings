# CodeHawks Escrow Contract - Competition Details - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
    - ### [M-01. Tokens utilizing a Blocklist, such as USDC, can hinder the Seller or Arbiter from accessing the funds](#M-01)
- ## Low Risk Findings
    - ### [L-01. Absence of access control verification in the arbiter address](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Cyfrin

### Dates: Jul 24th, 2023 - Aug 5th, 2023

[See more contest details here](https://www.codehawks.com/contests/cljyfxlc40003jq082s0wemya)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 0
   - Medium: 1
   - Low: 1



		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Tokens utilizing a Blocklist, such as USDC, can hinder the Seller or Arbiter from accessing the funds            



## Summary

USDC token has implemented an administratively owned blocklist feature, effectively restricting users from transmitting or acquiring funds within this Token.

## Vulnerability Details

The vulnerability may arise when the SELLER or ARBITER is unaware of being included in the USDC blocklist, before initiating an Escrow. The escrow contract does not currently incorporate the proper verifications to mitigate this specific blocklist scenario, leading to funds becoming stucked within the Escrow.

This vulnerability holds significant importance due to the widespread use of the USDC token and its intended application in Escrows. As such, careful attention must be given to addressing and mitigating this issue effectively.

## Impact

Due to the lack of verification for the Seller's or Arbiter's presence on the blocklist, the Escrow might be unable to disburse funds to them.

## Tools Used

Static review

## Recommendations

To address this vulnerability, it is advisable to implement a check within the constructor of the Escrow contract, specifically verifying whether the Token used in the Escrow is USDC and consequently checking for blocklist.



# Low Risk Findings

## <a id='L-01'></a>L-01. Absence of access control verification in the arbiter address            



## Summary

In Escrow.sol contract there is no mechanism (in the constructor or possibly a modifier) to control that buyer or seller cannot be assigned the role or arbiter. This simple lack of verification has direct impact on the implementation roles and the services offered in the whole contract and its workflow.

## Vulnerability Details
The absence of arbiter's verification can lead that seller or buyer might assume this role. The details of this issue can be described in the following statements:

1) Firstly, all the three roles of Escrow.sol contract have stipulated independent functionalities, thus they must not accept two roles being assigned to the same address in accordance to the logic of the workflow, if so, it will be a clear violation of the role implementation. Hence this separation of entities is crucial since the deployment and should be handled correctly by the contract itself, aside from the designations or purpose each role may have.

2) Speaking of the role's purpose, while it is clearly stipulated that arbiter is a trusted role and "If the buyer accidentally or maliciously deploys an Escrow with incorrect arbiter details, then the seller could refuse to provide their services" and overalls the role should be "Impartial", this lack of verification of the arbiter role, could allow a buyer or seller to be arbiter themselves, having one role in control of more than half of the workflow/functionalities, something than can be corrected directly inside the codebase, and not offchain. The current implementation leaves this potential vector completely opened, in such a "centralized way" depending solely in the good faith of the buyer.

3) In relation to the codebase, if this verification is not in place and a buyer or seller has assigned the arbiter role, the modifier onlyBuyerOrSeller() is shadowed completely. In consequence, an arbiter can call directly the function to initiate a dispute, when this not part of its role, which is an access control violation to the contract designations.

The last asseveration naturally breaks the workflow with a seller or buyer controlling the arbiter role, and the services/funds provided.

## Impact
If a seller or buyer has the arbiter role they can initiate and resolve disputes in a way that does not correspond to their respective designations and control the contract's workflow.

## Tools Used
Manual Review

## Proof of concept

For the sake of the demonstration, I assigned momentarily to the EscrowTestBase.t.sol the SELLER address to the ARBITER, and proceeded to check that only buyer or seller can initiate dispute, with the ARBITER as the caller of the function test.

```
contract EscrowTestBase {
    bytes32 public constant SALT1 = bytes32(uint256(keccak256(abi.encodePacked("test"))));
    bytes32 public constant SALT2 = bytes32(uint256(keccak256(abi.encodePacked("test2"))));
    uint256 public constant PRICE = 1e18;
    IERC20 public immutable i_tokenContract;
    address public constant BUYER = address(1);
    address public constant SELLER = address(2);
    //address public constant ARBITER = address(3);
    address public constant ARBITER = SELLER;
    uint256 public constant ARBITER_FEE = 1e16;
}

function testOnlyBuyerOrSellerCanCallinitiateDispute(address randomAdress) public escrowDeployed {

        /*
        vm.assume(randomAdress != BUYER && randomAdress != SELLER);
        vm.expectRevert(IEscrow.EscrowOnlyBuyerOrSeller.selector);
        vm.prank(randomAdress);
        escrow.initiateDispute();
        */


        vm.prank(ARBITER);
        vm.expectRevert(IEscrow.EscrowOnlyBuyerOrSeller.selector); 
        escrow.initiateDispute();

    }
```

The function above expects to be reverted because only buyer or seller, not an arbiter, can initiate a dispute. Nevertheless, the test fails because the arbiter is seen as a seller, which we deliberately assigned in the beginning of the scenario shadowing the onlyBuyerOrSeller() modifier.

## Recommendations

Implement a verification in codebase (Escrow.sol) ensuring that an arbiter cannot be a seller or a buyer to minimize the potential exploitation of the arbiter role offchain.



