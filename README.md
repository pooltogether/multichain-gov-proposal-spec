# Multi-chain Governance

This repository defines API specifications for a multi-chain governance system. This governance interface allowß protocols to easily capture and broadcast state updates across multiple L2s or sidechains.

The specification is broken into two parts: proposals and voting. Splitting proposals from voting affords flexibility in the governance implementation.

## Proposals

The [proposal specification](./MultiChainProposals.md) defines the structure of proposals and how they are created. This allows us to standardize the proposal creation interface as well as the structure of events.

## Voting

The [voting specification](./MultiChainVoting.md) defines a voting interface that makes it easy to add this governance to existing tokens while mitigating the problem of clock drift.