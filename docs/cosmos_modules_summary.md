# Cosmos SDK Modules Summary

This document provides a summary of the most common Cosmos SDK modules found in the `x/` directory path.

## Core Modules

| Module | Function |
|--------|----------|
| `x/auth` | Manages accounts, signatures, and transaction authentication |
| `x/bank` | Handles token transfers and balance tracking |
| `x/staking` | Manages validator delegation, bonding, and proof-of-stake consensus |
| `x/distribution` | Distributes staking rewards and fees to validators and delegators |
| `x/slashing` | Penalizes validators for misbehavior like double-signing or downtime |
| `x/gov` | Enables on-chain governance proposals and voting mechanisms |
| `x/mint` | Controls token inflation and block reward minting |

## Additional Common Modules

| Module | Function |
|--------|----------|
| `x/params` | Manages global chain parameters that can be modified via governance |
| `x/upgrade` | Handles coordinated blockchain software upgrades |
| `x/evidence` | Processes and stores evidence of validator misbehavior |
| `x/crisis` | Provides emergency halt mechanisms when invariants are broken |
| `x/genutil` | Utilities for genesis block creation and validation |
| `x/capability` | Manages object-capability-based access control for IBC |
| `x/ibc` | Implements Inter-Blockchain Communication protocol for cross-chain transfers |
| `x/transfer` | Handles IBC token transfers between different blockchains |
| `x/feegrant` | Allows accounts to pay transaction fees on behalf of others |
| `x/authz` | Provides authorization mechanisms for account-to-account permissions |
| `x/group` | Enables multi-signature accounts and group decision making |
| `x/nft` | Basic non-fungible token functionality and operations |

## Notes

These modules form the foundational layer that most Cosmos-based blockchains build upon, with many chains adding custom modules in their own `x/` directories for specific features.