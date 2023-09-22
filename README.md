# Filecoin Storage Bounty

Filecoin Storage Bounty is an EVM contract built on Filecoin's FEVM to be used for making bounty-based payments to Filecoin Storage Providers for storing data specified in Storage Bounties.

Lifecycle for a bounty:
1. Bounty creator creates a storage bounty for a list of Piece CIDs.
2. Bounty sponsor deposits the bounty reward into escrow for the bounty.
3. SP seals and stores data with Filecoin for all of the Piece CIDs.
4. SP submits deal IDs containing all of the bounty's Piece CIDs and collects the bounty reward from escrow.

# Domain Terminology

**SP**: Storage Provider

**Piece CID**: Content Identifier for the piece commitment of data to be stored with Filecoin

# Architecture

The general architecture of this contract is a singleton-style contract that manages multiple storage bounties in a single contract deployement. This was chosen over a "contract per bounty" architecture to avoid the gas overhead of deploying the contract's bytecode for every bounty. This gas is saved at the cost of more complexity and security considerations revolving around the contract managing token balances and permissions for multiple bounties concurrently.

## Notes

- The data types defined in this spec are meant to represent funcionality of the types. They are not meant to specify any implementation details. For instance some of the struct arrays in these data types should not be actual arrays of structs, but should be integer arrays corresponding to indices in a mapping holding those struct instances. In addition function types defined here are only meant to communicate functionality, for instance no thought is given in the spec between `external` and `public` and many arguments are missing memory properties.
- Related to previous point, there should be one single mapping for all `Eligibility` instances, that can be referenced by both `bountyCreatorEligibility` in the `GlobalConfig` and `spElibiligity` in `BountyStorageConfig`. 
- This contract supports using both native FIL and/or any ERC20 token as bounty rewards. Any place the `address` type is used to identify a token denomination, it's value should be either a valid ERC20 contract address or `address(0)` to indicate native FIL.

## Contract Types

### `GlobalConfig`

| Parameter                | Type          | Description                                                                       |
| ------------------------ | ------------- | --------------------------------------------------------------------------------- |
| bountyCreatorEligibility | `Eligibility` | Eligibility requirements for accounts to create a bounty.                         |
| minReplication           | `uint32`      | Minimum accepted `replicationFactor` when creating bounties.                      |
| maxReplication           | `uint32`      | Maximum accepted `replicationFactor` when creating bounties. (`0` for no maximum) |

### `Eligibility`

| Parameter       | Type              | Description                                                                                                                                                                                                                                                                                                                      |
| --------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| eligibilityType | `EligibilityType` | The type of eligibility this instance represents. Enum with the following values:<br/>**`ACCOUNTS`**: Allowlist of account addresses<br/>**`ERC20`**: Allowlist of ERC20 tokens<br/>**`ERC721`**: Allowlist of ERC721 tokens<br/>**`AND`**: Recursive AND<br/>**`OR`**: Recursive OR<br/>**`OPEN`**: No eligibility requirements |
| addressList     | `address[]`       | List of addresses for eligibility.<br/>**`ACCOUNTS`**: SP eligible if it has an account on this address list<br/>**`ERC20`**: SP eligible if it holds ERC20 tokens specified by this address list<br/>**`ERC721`**: SP eligible if it holds an ERC721 token on this address list<br/>**`AND` `OR` `OPEN`**: unused               |
| param1          | `uint256`         | General parameter 1.<br/>**`ERC20`**: minimum balance of ERC20 token for SP eligibility<br/>**`ACCOUNTS` `ERC721` `AND` `OR` `OPEN`**: unused                                                                                                                                                                                    |
| param2          | `Eligibility[]`   | General parameter 2.<br/>**`AND`**: SP eligibile if **all** eligibility requirements in this list are satisfied<br/>**`OR`**: SP eligibile if **any** eligibility requirements in this list are satisfied<br/>**`ACCOUNTS` `ERC20` `ERC721` `OPEN`**: unused                                                                     |

### `StorageBountyConfig`

| Parameter         | Type                    | Description                                                                                                                                                                                                                                                                                                     |
| ----------------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| creator           | `address`               | Creator of the bounty, sets the bounty config parameters during bounty creation.                                                                                                                                                                                                                                |
| sponsor           | `address`               | Sponsor funding the bounty. Can be the same as `creator` or can be a different address. This is the **only** account that can deposit into the escrow balance for the bounty or withdraw from the escrow balance if the bounty expires without being collected.                                                 |
| rewards           | `StorageBountyReward[]` | List of financial rewards for the bounty. This is a list to enable a bounty to include more than one token type or include multiple destinations for those tokens.                                                                                                                                              |
| startEpoch        | `int64`                 | Start epoch for the bounty. Bounty reward can't be collected before this time.                                                                                                                                                                                                                                  |
| expireEpoch       | `int64`                 | Expiration epoch for the bounty. If bounty has not been fulfilled by this time, the bounty expires and can no longer be fulfilled. Bounty balance now available to be withdrawn by Sponsor.                                                                                                                     |
| data              | `string[]`              | List of Piece CIDs for prepared data. These are <=32Gb IPLD chunks that must be stored in storage deals for an SP to collect the bounty.<br/><br/>Note: All Piece CIDs must be stored by an SP before they can collect the reward. An SP can't collect X% of reward after storing X% of Piece CIDs.             |
| spEligibility     | `Eligibility`           | Eligibility requirements for SPs to be able to fulfill and collect the bounty.                                                                                                                                                                                                                                  |
| replicationFactor | `uint32`                | Replication required before bounty can be paid out.<br/>Examples:<br/>**`replicationFactor = 1`**: no replication required<br/>**`replicationFactor = 4`**: 4 different SPs must all store the data in the bounty before any payout is allowed, they each collect their 1/4 of the bounty reward independently. |

### `StorageBountyReward`

| Parameter   | Type      | Description                                                                                                                                                                            |
| ----------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| token       | `address` | Address of ERC20 token for this reward. (or `address(0)` for native FIL)                                                                                                               |
| amount      | `uint256` | Amount of the specified token.                                                                                                                                                         |
| destination | `address` | Where the reward is transferred to when the bounty is collected. This can be set to `address(0)` to allow the collector of the bounty specify a destination at the time of collection. |

### `StorageBountyState`

| Parameter      | Type                           | Description                                                      |
| -------------- | ------------------------------ | ---------------------------------------------------------------- |
| escrowBalances | `StorageBountyEscrowBalance[]` | List of balances currently held by the contract for this bounty. |
| wasCollected   | `bool`                         | Set to true when the bounty is collected.                        |

### `StorageBountyEscrowBalance`

| Parameter | Type      | Description                                                 |
| --------- | --------- | ----------------------------------------------------------- |
| token     | `address` | Address of an ERC20 token. (or `address(0)` for native FIL) |
| amount    | `uint256` | Amount of the specified token.                              |

## Public Functions (write)

### `constructor` (+ supporting functions to configure contract)

```solidity
constructor()
    external
{
    ...
}
```

```solidity
function setBountyCreatorEligibility(
    uint256 bountyCreatorEligibilityId
)
    external
{
    ...
}
```

```solidity
function setMinReplication(
    uint32 minReplication
)
    external
{
    ...
}
```

```solidity
function setMaxReplication(
    uint32 maxReplication
)
    external
{
    ...
}
```

### `addEligibilityDefinition`

Add an eligibility definition to the global list held by the contract. This id can then be used to set `GlobalConfig.bountyCreatorEligiblity` or `StorageBountyConfig.spEligiblity`.

```solidity
function addElibilityDefinition(
    EligibilityType eligibilityType,
    address[] addressList,
    uint256 param1,
    uint256[] param2
)
    external
    returns (uint256 eligiblityId)
{
    ...
}
```

### `addRewardDefinition`

Add a reward definition to the global list held by the contract. These ids can then be used to set `StorageBountyConfig.rewards`.

```solidity
function addBountyRewardDefinition(
    address token,
    uint256 amount,
    address destination
)
    external
    returns (uint256 rewardId)
{
    ...
}
```

```solidity
function addBountyRewardDefinitions(
    address[] token,
    uint256[] amount,
    address[] destination
)
    external
    returns (uint256 rewardIds)
{
    ...
}
```

### `createBounty`

Create a new bounty.

- Can only be done by accounts satisfying `bountyCreatorEligibility` in the global config of the deployed contract.

```solidity
function createBounty(
    address sponsor,
    uint256[] rewardIds,
    int64 startEpoch,
    int64 expireEpoch,
    string[] data,
    uint256 spEligibilityId,
    uint32 replicationFactor
)
    external
    returns (uint256 bountyId)
{
    ...
}
```

### `sponsorBounty`

Sponsor a bounty.

- Can only be done by `sponsor` of the bounty.
- Can only be done if the the current block epoch is less than `expireEpoch`.
- Needs to be called separately for each different token denomination included in the rewards list for the bounty.
- For ERC20 tokens, the UI should also provide an interface and/or prompt for a sponser to transact directly with the ERC20 contract to set allowance for this contract.

```solidity
function sponsorBounty(
    uint256 bountyId,
    address token,
    uint256 amount
)
    external
{
    ...
}
```

### `withdrawBounty`

Withdraw a bounty.

- This withdraws all rewards for the bounty back to the `sponsor`'s account.
- Can only be done by `sponsor` of the bounty.
- Can only be done if the the current block epoch is greater than `expireEpoch` in the bounty and the bounty was not collected.

```solidity
function withdrawBounty(
    uint256 bountyId,
    ...TODO
)
    external
{
    ...
}
```

### `collectBounty`

Collect a bounty.

- Can only be done by accounts satisfying `spEligiblity` in the bounty.
- Can only be done when the current block epoch is between `startEpoch` and `expireEpoch` in the bounty.
- The SP must provide `dealIds` containing a list of active storage deals that fulfill all the piece commitments listed in `data` in the bounty config.
- If any reward destinations are unspecified by the creator of the bounty, they will go to `rewardDestination` provided by the SP. If all reward destinations are specified, then this argument doesn't affect anything.

```solidity
function collectBounty(
    uint256 bountyId,
    string[] dealIds,
    address, rewardDestination,
)
    external
{
    ...
}
```

## Public Functions (read)

### Typical CRUD access

Typical read functions for getting list of bounties that exist and bounty info by id.

```solidity
...

{read-only CRUD functions}

...
```

### `checkAccountIsEligibleToCreateBounty`

Check if an account is eligible to create a bounty using this contract deployment.

```solidity
function checkAccountIsEligibleToCreateBounty(
    addresss account
)
    external
    return (bool accountIsEligible)
{
    ...
} 
```

### `checkAccountIsEligibleToFulfillBounty`

Check if an account is eligible to fulfill a specific bounty. This bounty only checks an account against the `spEligibility` config for the bounty. It does not take into account any other state of the bounty such as `startEpoch` and `expireEpoch` times or whether the bounty has been sponsored or even already fulfilled and collected.

```solidity
function checkAccountIsEligibleToFulfillBounty(
    addresss account,
    uint256 bountyId
)
    external
    return (bool accountIsEligible)
{
    ...
} 
```

### `checkDealsFulfillBounty`

Check if a list of deals fulfill a bounty.

- This is done by checking all the deal ids against the Storage Market Actor (built-in FVM actor)
  - [getDealDataCommitment](https://github.com/Zondax/filecoin-solidity/blob/master/contracts/v0.8/MarketAPI.sol#LL70C14-L70C35) to check against Piece CIDs in `data` list
  - [getDealActivation](https://github.com/Zondax/filecoin-solidity/blob/master/contracts/v0.8/MarketAPI.sol#LL162C14-L162C31) to check that deal has been activated by SP
  - NOTE: links above are to zondax sdk that provides solidity interface to the built-in FVM actors, but this could be forked for improved api design and usability if needed (`getDealActivation` can revert tx when really it should return an error code)

```solidity
function checkDealsFulfillBounty(
    uint256 bountyId,
    string[] dealIds,
)
    external
    returns (bool dealsFulfillBounty)
{
    ...
}
```

## Events

The contract has sufficient events such that the complete contract state can at all times be constructed from the event log.

## Errors

The contract returns useful errors whenever a transaction is purposefully failed by the contract logic (bad eligibility, invalid arguments, unmet bounty requirements, etc)

## Factory

There should be a separate factory contract capable of deploying and configuring some common configurations.

```solidity
function deployWithOpenBountyCreatorEligibility(
    uint32 minReplication,
    uint32 maxReplication
)
    external
    returns (address deployedContract)
{
    ...
}
```

```solidity
function deployWithBountyCreatorAllowList(
    address[] allowList,
    uint32 minReplication,
    uint32 maxReplication
)
    external
    returns (address deployedContract)
{
    ...
}
```

```solidity
function deployWithBountyCreatorNftRequirement(
    address[] erc721Contracts,
    uint32 minReplication,
    uint32 maxReplication
)
    external
    returns (address deployedContract)
{
    ...
}
```

```solidity
function deployWithBountyErc20Requirement(
    address[] erc20Contracts,
    uint256 minTokenAmount,
    uint32 minReplication,
    uint32 maxReplication
)
    external
    returns (address deployedContract)
{
    ...
}
```
