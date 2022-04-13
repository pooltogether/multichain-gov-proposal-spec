# Multi-Chain Governance Voting

# Abstract

This specification is an extension to [Multi-Chain Governance Proposals](./MultiChainProposals.md) that affords the ability to vote using standard ERC20 tokens. This specification mitigates security risks arising from clock drift between chains.

# Motivation

Many protocols need to govern smart contracts across Ethereum, Ethereum L2s, and EVM-compatible sidechains. This specification aims to standardize a multi-chain voting specification so that DAOs can leverage common infrastructure.

Typically protocols will bridge their token from Ethereum to other L2s and sidechains using whitelabel ERC20 bridges. This means that the token on the L2 or sidechain will be a plain ERC20. To add voting capabilities to the token it needs to be wrapped in a contract.

The locking behaviour defined in this spec mitigates a problem with cross-chain state: clock drift.  L2s and sidechains will not always share the exact same timestamp, so it's possible for a user to bridge their tokens and have their token balance reflected on two chains with the same timestamp. The locking behaviour defined in this spec mitigates that problem.

# Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

This specification introduces the VoteLocker contract, and adds additional features to the GovernorBranch and the GovernorRoot contracts.

## Definitions

**ProposalInfo**

```solidity
struct ProposalInfo {
    uint rootNonce;
    uint startTime;
    uint endTime;
    uint branchChainId;
    address branchAddress;
    uint branchNonce;
    bytes32 callHash;
}
```

**ProposalState**

```solidity
enum ProposalState {
    Active,
    Ended,
    Defeated,
    Succeeded
}
```

## VoteLocker

The VoteLocker contract allows users to deposit tokens for voting power. The deposit is locked for a duration specified by the user.

### Methods

**token**

Returns the address of the ERC20 token that users deposit.

```yaml
- name: token
  type: function
  stateMutability: view
  outputs:
    - name: token
      type: address
```

**lockVotes**

Allows a user to lock their voting tokens for a duration of time. Transfers `amount` of tokens from the `msg.sender` into the VoteLocker. Increases `msg.senders` balance.

MUST emit the `LockedVotes` event.

MUST revert if the user already has votes locked.

```yaml
- name: lockVotes
  type: function
  stateMutability: nonpayable
  inputs: 
    - name: amount
      type: uint256
    - name: unlockTime
      type: uint256
```

**increaseUnlockTime**

Increases the unlock time for a user.

MUST emit the `IncreasedUnlockTime` event.

```yaml
- name: increaseUnlockTime
  type: function
  stateMutability: nonpayable
  inputs: 
    - name: unlockTime
      type: uint256
```

**depositTo**

Deposits tokens for a user. Transfers tokens into the contract and increases the `to` users balance.

MUST revert if the `to` address does not have votes locked.

MUST emit the `Deposited` event.

```yaml
- name: depositTo
  type: function
  stateMutability: nonpayable
  inputs: 
    - name: to
      type: address
    - name: amount
      type: uint256
```

**pastVotes**

Retrieves a users voting power at `timestamp`.

Conceptually, the voting power of a user should be the smallest balance that they held between `timestamp - range` and `timestamp + range`. This ensures that any movement of tokens is ignored; only tokens that have been locked for the entire duration are eligible.  Since this API only allows locked deposits to increase, the smallest balance will always be at `timestamp - range`.

The users `unlock time` must be greater than or equal to `timestamp + range`. Note that it's possible for a user to extend their unlock time in order to vote on a proposal.

```yaml
- name: pastVotes
  type: function
  stateMutability: view
  inputs: 
    - name: timestamp
      type: uint256
```

**pastBalanceOf**

Retrieves a users balance at the given `timestamp`.

```yaml
- name: pastBalanceOf
  type: function
  stateMutability: view
  inputs: 
    - name: account
      type: address
    - name: timestamp
      type: uint256
  outputs:
    - name: balance
      type: uint256
```

**balanceOf**

Retrieves a users current balance of voting tokens.

```yaml
- name: balanceOf
  type: function
  stateMutability: view
  inputs: 
    - name: account
      type: address
  outputs:
    - name: balance
      type: uint256
```

**range**

Returns the range of time in seconds that users must hold tokens around a given timestamp. When determining voting power for a timestamp, this contract will look at the range `timestamp - range` to `timestamp + range`.

**withdraw**

Withdraws a users tokens and rescinds past and current voting power. The user should complete any voting before withdrawing.

MUST revert if the current time is greater than or equal to the users `unlock time`.

MUST emit the `Withdrawn` event.

### Events

**LockedVotes**

Emitted when a user locks their tokens in to vote.

```yaml
- name: LockedVotes
  type: event
  inputs:
    - name: account
      type: address
    - name: amount
      type: uint256
    - name: unlockTime
      type: uint256
```

**IncreasedUnlockTime**

Emitted when a user increases their lock time

```yaml
- name: IncreasedUnlockTime
  type: event
  inputs:
    - name: account
      type: address
    - name: unlockTime
      type: uint256
```

**Deposited**

Emitted when an account is deposited into.

```yaml
- name: Deposited
  type: event
  inputs:
    - name: from
      type: address
    - name: to
      type: address
    - name: amount
      type: uint256
```

**Withdrawn**

Emitted when a user withdraws their tokens.

```yaml
- name: Withdrawn
  type: event
  inputs:
    - name: account
      type: address
    - name: amount
      type: uint256
```

## GovernorBranch

### Methods

**castVote**

Allows a user to vote for a proposal. The proposal info is passed so that the proposal hash can be calculated for verification. The `support` options are: 0 = Against, 1 = For, 2 = Abstain. The users vote will be added to the appropriate support total for that proposal.

MUST NOT allow a user to vote twice for a proposal.

MUST emit the `VoteCast` event.

MUST revert if the current time is greater than the proposal `endTime`.

MUST revert if `support` is not one 0, 1 or 2.

```yaml
- name: castVote
  type: function
  stateMutability: nonpayable
  inputs: 
    - name: support
      type: uint8
    - name: proposal
      type: ProposalInfo
```

### Events

**VoteCast**

Emitted when a user casts their vote.

```yaml
- name: VoteCast
  type: event
  inputs:
    - name: proposalHash
      type: bytes32
    - name: voter
      type: address
    - name: support
      type: uint8
    - name: votes
      type: uint256
```

## GovernorRoot

The GovernorRoot contract has vote "aggregators", 

### Methods

**addVotes**

Add votes to the root from an aggregator.

MUST emit the `VotesAdded` event.

SHOULD only be callable by an authorized aggregator (such as a GovernorBranch or bridge).

```yaml
- name: addVotes
  type: function
  stateMutability: nonpayable
  inputs:
    - name: branchChainId
      type: uint256
    - name: branchAddress
      type: address
    - name: yesVotes
      type: uint256
    - name: noVotes
      type: uint256
    - name: abstainVotes
      type: uint256
    - name: proposalHash
      type: bytes32
```

**gracePeriod**

The grace period is the amount of time after a proposal ends during which aggregators can submit their votes.

```yaml
- name: gracePeriod
  type: function
  stateMutability: view
  outputs:
    - name: duration
      type: uint256
```

**proposalStatus**

Returns the status of a proposal:

- Active: the proposal has not yet ended
- Ended: the proposal has ended but the votes are not yet in
- Succeeded: voting has completed and the proposal has passed
- Defeated: voting has completed and the proposal did not pass

```yaml
- name: proposalStatus
  type: function
  stateMutability: view
  inputs:
    - name: proposalHash
      type: bytes32
  outputs:
    - name: status
      type: ProposalState
```

### Events

**VotesAdded**

Emitted when the final vote count comes in for a GovernorBranch

```yaml
- name: VotesAdded
  type: event
  inputs:
    - name: branchChainId
      type: uint256
    - name: branchAddress
      type: address
    - name: yesVotes
      type: uint256
    - name: noVotes
      type: uint256
    - name: abstainVotes
      type: uint256
    - name: proposalHash
      type: bytes32
```

# Rationale

Prevents double voting across chains.

*notes* the act of bridging passed proposals can be done as-needed for GovernorBranches that need to execute transactions.

## Flow Diagram

![](./assets/GovernanceVoting.png)

# Backwards Compatibility

This implementation takes inspiration from Curve Vote Escrow contracts, and from the OpenZeppelin ERC20Vote extensions. However, it is not compatible with those specifications.

# Test Cases

TBD

# Reference Implementation

TBD

*Implementation Note: I'm guessing we can just store an array of (balance, timestamp) tuples per user and binary search them to find the balance. Deposits push onto array, withdrawal clears the array*

# Security Considerations

This specification requires users to hold a minimum amount of tokens for a time range. This time range should be configured to exceed the expected maximum clock drift between chains.
