# CosmWasm Security Guide

## CosmWasm Contract Reply Message States

CosmWasm contracts can handle replies from submessages using different reply modes that control when and how the contract receives callback information.

### Reply Message States

**ReplyAlways**
- The contract receives a reply callback regardless of whether the submessage execution succeeds or fails
- Provides full control over handling both success and error cases
- The reply handler can access the submessage result and decide how to proceed

**ReplyOnSuccess**
- Reply callback is only triggered when the submessage executes successfully
- No callback occurs if the submessage fails
- Useful when you only care about successful outcomes

**ReplyOnError**
- Reply callback is only triggered when the submessage execution fails
- No callback occurs if the submessage succeeds
- Useful for error handling and recovery scenarios

**ReplyNever**
- No reply callback is sent to the contract
- Fire-and-forget execution model
- The contract cannot handle the submessage result

### Security Implications

**State Management Risks**
- With ReplyAlways and ReplyOnSuccess, failed submessages can leave your contract in an inconsistent state if not handled properly
- You must carefully manage state transitions in reply handlers to avoid partial updates

**Error Handling Vulnerabilities**
- ReplyOnError allows you to implement custom error recovery, but improper handling can mask critical failures
- ReplyNever provides no feedback, so you can't detect or respond to failures

**Reentrancy Concerns**
- Reply handlers are called after submessage execution, creating potential reentrancy attack vectors
- State should be carefully managed between submessage dispatch and reply handling
- Consider using commit-reveal patterns for sensitive operations

**Gas Exhaustion**
- Reply handlers consume additional gas, and complex reply logic can lead to out-of-gas failures
- Failed reply handlers can cause the entire transaction to revert

**Access Control**
- Reply handlers bypass normal message authentication since they're triggered by the system
- Ensure reply handlers properly validate the submessage ID and context
- Don't rely on reply handlers for security-critical state changes without proper validation

**Best Practices**
- Use ReplyAlways for critical operations where you need guaranteed feedback
- Implement proper error recovery in reply handlers
- Validate submessage IDs in reply handlers to prevent confusion
- Keep reply handler logic simple to minimize gas costs and attack surface
- Consider state rollback mechanisms for failed submessages

The choice of reply mode significantly impacts your contract's security posture and error handling capabilities.

## Commit-Reveal Patterns

Commit-reveal patterns are cryptographic schemes used to prevent front-running and ensure fairness in blockchain applications.

### How Commit-Reveal Works

**Phase 1: Commit**
- Participants submit a cryptographic hash of their intended action plus a secret nonce
- The actual action/value remains hidden until the reveal phase
- Example: `hash(bid_amount + secret_nonce)` for an auction

**Phase 2: Reveal**
- After the commit phase ends, participants reveal their original values and nonces
- The contract verifies that `hash(revealed_value + revealed_nonce)` matches the committed hash
- Only valid reveals are accepted and processed

### Security Benefits

**Front-Running Prevention**
- Other participants can't see your intended action during the commit phase
- Prevents MEV (Maximal Extractable Value) attacks where bots copy profitable transactions
- Essential for auctions, voting, and competitive scenarios

**Fairness Guarantee**
- All participants commit simultaneously without knowing others' actions
- Eliminates information asymmetry that could be exploited
- Ensures decisions are made independently

**Timing Attack Resistance**
- Separates decision-making from execution timing
- Prevents late participants from gaining unfair advantages
- Reduces the impact of network latency differences

### CosmWasm Implementation Example

```rust
// Commit phase
pub fn commit_bid(
    deps: DepsMut,
    info: MessageInfo,
    commitment: String, // hash(bid_amount + nonce)
) -> Result<Response, ContractError> {
    let commit = Commitment {
        bidder: info.sender.clone(),
        commitment: commitment.clone(),
        block_height: env.block.height,
    };
    COMMITMENTS.save(deps.storage, &info.sender, &commit)?;
    Ok(Response::new())
}

// Reveal phase
pub fn reveal_bid(
    deps: DepsMut,
    info: MessageInfo,
    bid_amount: Uint128,
    nonce: String,
) -> Result<Response, ContractError> {
    let commitment = COMMITMENTS.load(deps.storage, &info.sender)?;
    
    // Verify the commitment
    let expected_hash = hash(bid_amount.to_string() + &nonce);
    if expected_hash != commitment.commitment {
        return Err(ContractError::InvalidReveal {});
    }
    
    // Process the revealed bid
    process_bid(deps, info.sender, bid_amount)
}
```

### Common Use Cases

**Auctions**
- Sealed bid auctions where bids are hidden until reveal
- Prevents bid sniping and ensures fair price discovery

**Voting Systems**
- Hide votes until voting period ends
- Prevents vote manipulation and coercion

**Random Number Generation**
- Collect entropy from multiple participants
- Prevents predictable or manipulated randomness

**Token Sales/ICOs**
- Hide purchase amounts to prevent market manipulation
- Ensures fair allocation mechanisms

### Implementation Considerations

**Time Windows**
- Clearly defined commit and reveal periods
- Automatic phase transitions to prevent manipulation

**Penalty Mechanisms**
- Punish participants who commit but don't reveal
- Typically involves forfeiting deposits or fees

**Hash Function Security**
- Use cryptographically secure hash functions (SHA-256, etc.)
- Ensure sufficient entropy in nonces to prevent brute force attacks

**State Management**
- Properly clean up commitments after reveals
- Handle edge cases like late reveals or invalid commitments

Commit-reveal patterns are essential for creating fair, transparent, and manipulation-resistant decentralized applications, especially in competitive or sensitive contexts.

## Transaction Reversion Behavior

### Default Behavior

**ReplyOnSuccess**
- If the submessage fails, **the entire transaction reverts by default**
- No reply callback is triggered since there's no success to report
- The contract has no opportunity to handle the failure

**ReplyNever**
- If the submessage fails, **the entire transaction reverts by default**
- No reply callback exists to handle the failure
- The contract cannot recover from or respond to the failure

### Preventing Transaction Reversion

You can prevent automatic reversion by using the `ReplyOn::Never` with specific submessage configuration:

```rust
// This will NOT revert the transaction on submessage failure
let submsg = SubMsg {
    id: 1,
    msg: your_message,
    gas_limit: Some(100_000),
    reply_on: ReplyOn::Never,
};

// But you won't know if it failed!
```

However, there's a better approach using **error handling with ReplyAlways**:

```rust
// This gives you control over failure handling
let submsg = SubMsg {
    id: 1,
    msg: your_message,
    gas_limit: Some(100_000),
    reply_on: ReplyOn::Always, // Handle both success and failure
};

// In your reply handler:
pub fn reply(deps: DepsMut, env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.result {
        SubMsgResult::Ok(response) => {
            // Handle success
            Ok(Response::new().add_attribute("submsg", "success"))
        }
        SubMsgResult::Err(error) => {
            // Handle failure without reverting the main transaction
            Ok(Response::new()
                .add_attribute("submsg", "failed")
                .add_attribute("error", error))
        }
    }
}
```

### Security Implications

**Silent Failures with ReplyNever**
- If you somehow prevent reversion, failures become silent
- Your contract might assume the submessage succeeded when it didn't
- This can lead to inconsistent state and security vulnerabilities

**Partial State Updates**
- With ReplyOnSuccess, failed submessages can leave your contract in a partially updated state
- You might have updated local state before the submessage, then the whole transaction reverts

**Best Practice Pattern**
```rust
// Update state only after submessage succeeds
pub fn execute_with_submsg(deps: DepsMut, env: Env, info: MessageInfo) -> Result<Response, ContractError> {
    // DON'T update state here - wait for reply confirmation
    
    let submsg = SubMsg {
        id: 1,
        msg: external_call,
        reply_on: ReplyOn::Always,
        gas_limit: Some(100_000),
    };
    
    Ok(Response::new().add_submessage(submsg))
}

pub fn reply(deps: DepsMut, env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.result {
        SubMsgResult::Ok(_) => {
            // NOW it's safe to update state
            update_contract_state(deps)?;
            Ok(Response::new())
        }
        SubMsgResult::Err(_) => {
            // Handle failure gracefully
            Ok(Response::new().add_attribute("status", "failed"))
        }
    }
}
```

### Summary

- **ReplyOnSuccess + failure = transaction reverts** (default behavior)
- **ReplyNever + failure = transaction reverts** (default behavior)
- **ReplyAlways** gives you control to handle failures without reverting
- Use ReplyAlways for critical operations where you need guaranteed feedback and error handling
- Always design your state management around the possibility of submessage failures

The key is understanding that submessage failures are propagated up by default, and you need explicit error handling (via ReplyAlways) to prevent transaction reversion.

## Missing Callbacks Edge Case

### When Callbacks Are Expected But Never Arrive

**ReplyOnSuccess, ReplyOnError, ReplyAlways**
- If the submessage executes but the reply callback is never delivered, your contract becomes **permanently stuck**
- The main transaction will **hang indefinitely** waiting for the reply
- This is a **blocking operation** - your contract cannot proceed without the callback

### Root Causes of Missing Callbacks

**Chain Halts/Consensus Issues**
- If the blockchain stops or has consensus problems after submessage execution
- Network partitions or validator failures
- Chain upgrades that interrupt message processing

**Gas Exhaustion**
- If the reply handler runs out of gas during execution
- The callback fails but the submessage already executed
- Can create inconsistent state

**Contract Bugs**
- Malformed reply handlers that panic or fail
- Infinite loops in reply processing
- Invalid state transitions

**Infrastructure Failures**
- CosmWasm runtime bugs or memory issues
- Node crashes during message processing
- IBC timeout scenarios for cross-chain messages

### Security and Operational Impacts

**Permanent Lock-Up**
```rust
// This pattern is dangerous!
pub fn risky_pattern(deps: DepsMut) -> Result<Response, ContractError> {
    // Contract state is now locked until reply arrives
    let submsg = SubMsg {
        id: 1,
        msg: external_call,
        reply_on: ReplyOn::Always,
        gas_limit: None, // No gas limit = potential hang
    };
    
    Ok(Response::new().add_submessage(submsg))
}
```

**Funds Trapped**
- Any funds locked in the contract become inaccessible
- User deposits or protocol funds may be permanently lost
- No way to recover without chain-level intervention

**Protocol Breakdown**
- Dependent contracts may also become stuck
- Cascading failures across DeFi protocols
- Cross-chain operations may fail

### Mitigation Strategies

**1. Implement Timeouts**
```rust
pub struct PendingOperation {
    submsg_id: u64,
    created_at: u64,
    timeout_height: u64,
    fallback_action: String,
}

pub fn execute_with_timeout(
    deps: DepsMut,
    env: Env,
) -> Result<Response, ContractError> {
    let timeout_height = env.block.height + TIMEOUT_BLOCKS;
    
    let pending = PendingOperation {
        submsg_id: 1,
        created_at: env.block.height,
        timeout_height,
        fallback_action: "refund".to_string(),
    };
    
    PENDING_OPERATIONS.save(deps.storage, 1, &pending)?;
    
    let submsg = SubMsg {
        id: 1,
        msg: external_call,
        reply_on: ReplyOn::Always,
        gas_limit: Some(200_000),
    };
    
    Ok(Response::new().add_submessage(submsg))
}

pub fn cleanup_stale_operations(
    deps: DepsMut,
    env: Env,
) -> Result<Response, ContractError> {
    let stale_ops: Vec<_> = PENDING_OPERATIONS
        .range(deps.storage, None, None, Order::Ascending)
        .filter(|item| {
            if let Ok((_, op)) = item {
                env.block.height > op.timeout_height
            } else {
                false
            }
        })
        .collect();
    
    for (id, op) in stale_ops {
        // Execute fallback action
        execute_fallback(deps.branch(), &op.fallback_action)?;
        PENDING_OPERATIONS.remove(deps.storage, id);
    }
    
    Ok(Response::new())
}
```

**2. Use Circuit Breakers**
```rust
pub fn emergency_stop(
    deps: DepsMut,
    info: MessageInfo,
) -> Result<Response, ContractError> {
    // Only admin can trigger
    ensure_admin(deps.as_ref(), &info.sender)?;
    
    // Cancel all pending operations
    PENDING_OPERATIONS.clear(deps.storage);
    
    // Enable emergency mode
    CONFIG.update(deps.storage, |mut config| -> Result<_, ContractError> {
        config.emergency_mode = true;
        Ok(config)
    })?;
    
    Ok(Response::new())
}
```

**3. Implement Heartbeat Mechanisms**
```rust
pub fn heartbeat(
    deps: DepsMut,
    env: Env,
) -> Result<Response, ContractError> {
    // Update last activity timestamp
    LAST_ACTIVITY.save(deps.storage, &env.block.height)?;
    
    // Check for stale operations
    cleanup_stale_operations(deps, env)
}
```

**4. Use Gas Limits**
```rust
// Always set reasonable gas limits
let submsg = SubMsg {
    id: 1,
    msg: external_call,
    reply_on: ReplyOn::Always,
    gas_limit: Some(100_000), // Prevent infinite gas consumption
};
```

### Best Practices

**Design for Failure**
- Always assume callbacks might never arrive
- Implement recovery mechanisms from day one
- Use timeouts for all submessage operations

**Monitor and Alert**
- Track pending operations and their durations
- Set up alerts for stuck transactions
- Monitor gas usage patterns

**Test Edge Cases**
- Simulate network failures in tests
- Test timeout and recovery scenarios
- Verify fallback mechanisms work correctly

**Consider Alternatives**
- Use ReplyNever for non-critical operations
- Implement asynchronous patterns instead of blocking calls
- Consider using events for loose coupling

The key insight is that **submessages with reply callbacks are synchronous blocking operations** that can permanently lock your contract if not handled properly. Always design with failure scenarios in mind.

## CosmWasm Contract Migration

CosmWasm contract migration is a powerful feature that allows upgrading contract logic while preserving state.

### How Migration Works

**Technical Process**
1. A new contract code is uploaded to the chain (getting a new code ID)
2. The migration transaction is submitted with the new code ID
3. The chain calls the `migrate()` function in the NEW contract code
4. The migrate function can read/modify the existing contract's state
5. The contract's code ID is updated to point to the new code
6. All future calls use the new contract logic

**Migration Function Structure**
```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn migrate(
    deps: DepsMut,
    _env: Env,
    msg: MigrateMsg,
) -> Result<Response, ContractError> {
    // Version checking
    let stored_version = get_contract_version(deps.storage)?;
    if stored_version.contract != CONTRACT_NAME {
        return Err(ContractError::InvalidMigration {});
    }
    
    // State migration logic
    match msg {
        MigrateMsg::UpdateConfig { new_param } => {
            migrate_config(deps.storage, new_param)?;
        }
        MigrateMsg::UpgradeStorageFormat {} => {
            migrate_storage_format(deps.storage)?;
        }
    }
    
    // Update contract version
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
    
    Ok(Response::new().add_attribute("action", "migrate"))
}
```

### Who Can Migrate

**Admin-Only Migration (Default)**
- Only the contract admin can initiate migration
- Admin is set during contract instantiation
- Most secure option for protocol-owned contracts

**Governance Migration**
- Chain governance can migrate contracts through proposals
- Requires community consensus
- Common for critical infrastructure contracts

**No Migration (Immutable)**
- Admin can be permanently removed during instantiation
- Contract becomes immutable - no future migrations possible
- Highest security but no upgrade path

**Examples of Authorization**
```rust
// During instantiation
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    // Set admin for migration capability
    match msg.admin {
        Some(admin) => {
            // Specific admin can migrate
            cw2::set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
        }
        None => {
            // No admin = immutable contract
            // Migration will be impossible
        }
    }
    
    Ok(Response::new())
}
```

### Security Implications

**Complete Code Replacement**
- New contract code has full access to existing state
- Can completely change contract behavior
- Essentially unlimited power over funds and logic

**State Corruption Risks**
```rust
// Dangerous migration example
pub fn migrate(deps: DepsMut, _env: Env, msg: MigrateMsg) -> Result<Response, ContractError> {
    // This could corrupt existing state if not careful
    let old_config: OldConfig = OLD_CONFIG.load(deps.storage)?;
    
    // Wrong: This might not handle all fields correctly
    let new_config = NewConfig {
        param1: old_config.param1,
        param2: old_config.param2,
        // Missing param3 - could cause issues
    };
    
    NEW_CONFIG.save(deps.storage, &new_config)?;
    Ok(Response::new())
}
```

**Admin Key Risks**
- Admin private key compromise = complete contract control
- Single point of failure for all contract security
- No way to revoke migration once initiated

**Governance Attacks**
- Malicious governance proposals could steal funds
- Voters might not understand technical implications
- Coordination attacks on governance systems

### Migration Patterns and Best Practices

**Version Management**
```rust
use cw2::{get_contract_version, set_contract_version};

pub fn migrate(deps: DepsMut, _env: Env, msg: MigrateMsg) -> Result<Response, ContractError> {
    let version = get_contract_version(deps.storage)?;
    
    match version.version.as_str() {
        "1.0.0" => migrate_from_v1_to_v2(deps.storage)?,
        "1.1.0" => migrate_from_v1_1_to_v2(deps.storage)?,
        _ => return Err(ContractError::UnsupportedVersion {}),
    }
    
    set_contract_version(deps.storage, CONTRACT_NAME, "2.0.0")?;
    Ok(Response::new())
}
```

**Safe State Migration**
```rust
// Safe migration pattern
pub fn migrate_storage_format(storage: &mut dyn Storage) -> Result<(), ContractError> {
    // Read old format
    let old_items: Vec<OldItem> = OLD_ITEMS
        .range(storage, None, None, Order::Ascending)
        .collect::<Result<Vec<_>, _>>()?;
    
    // Clear old storage
    OLD_ITEMS.clear(storage);
    
    // Convert and save in new format
    for (key, old_item) in old_items {
        let new_item = NewItem {
            id: old_item.id,
            data: old_item.data,
            timestamp: old_item.timestamp,
            // Add new fields with defaults
            new_field: "default_value".to_string(),
        };
        NEW_ITEMS.save(storage, key, &new_item)?;
    }
    
    Ok(())
}
```

**Gradual Migration**
```rust
// For large datasets, implement gradual migration
pub fn migrate_batch(
    deps: DepsMut,
    msg: MigrateBatchMsg,
) -> Result<Response, ContractError> {
    let start_key = msg.start_key;
    let limit = msg.limit.unwrap_or(100);
    
    // Migrate a batch of items
    let items: Vec<_> = OLD_ITEMS
        .range(storage, start_key, None, Order::Ascending)
        .take(limit)
        .collect::<Result<Vec<_>, _>>()?;
    
    for (key, item) in items {
        migrate_single_item(deps.storage, key, item)?;
    }
    
    Ok(Response::new().add_attribute("migrated", limit.to_string()))
}
```

### Migration Strategies

**Multi-Sig Admin**
- Use multi-signature wallets for admin control
- Require multiple parties to approve migrations
- Reduces single point of failure

**Timelock Migrations**
- Implement delays between migration proposal and execution
- Allow time for community review
- Enable emergency stops if issues are found

**Governance-Controlled**
- Transfer admin to governance contract
- Require community voting for migrations
- More decentralized but potentially slower

**Immutable After Launch**
- Remove admin after initial deployment phase
- Accept that bugs cannot be fixed
- Highest security assurance for users

### Testing Migrations

**Integration Tests**
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_migration_v1_to_v2() {
        let mut deps = mock_dependencies();
        
        // Setup v1 state
        setup_v1_state(&mut deps);
        
        // Execute migration
        let msg = MigrateMsg::UpgradeToV2 {};
        let res = migrate(deps.as_mut(), mock_env(), msg).unwrap();
        
        // Verify v2 state is correct
        verify_v2_state(&deps);
    }
}
```

**Mainnet Testing**
- Test migrations on testnets first
- Use similar data volumes to mainnet
- Verify all edge cases work correctly

Migration is a double-edged sword - it enables upgrades and bug fixes but introduces significant security risks. The key is implementing appropriate governance and safety mechanisms based on your specific use case and risk tolerance.

## Native Token vs CW20 Security

### Native Token Security

**Checking `funds.denom` is SAFE for native tokens**
```rust
pub fn execute_deposit(
    deps: DepsMut,
    info: MessageInfo,
    env: Env,
) -> Result<Response, ContractError> {
    // This is SAFE - native tokens cannot be faked
    let payment = info.funds.iter()
        .find(|coin| coin.denom == "uatom")
        .ok_or(ContractError::InvalidPayment {})?;
    
    // The chain runtime guarantees this is real ATOM
    process_atom_deposit(deps, payment.amount)?;
    Ok(Response::new())
}
```

**Why it's safe:**
- Native token denoms are managed by the chain runtime
- Only the chain itself can mint/burn native tokens
- Smart contracts cannot create fake native tokens
- The `info.funds` field is populated by the chain, not user input

### CW20 Token Risks

**CW20 tokens have NO denom protection**
```rust
// DANGEROUS - This is NOT secure!
pub fn execute_cw20_deposit(
    deps: DepsMut,
    info: MessageInfo,
    env: Env,
    msg: Cw20ReceiveMsg,
) -> Result<Response, ContractError> {
    // Multiple CW20 contracts can have the same symbol!
    let token_info: TokenInfoResponse = deps.querier.query_wasm_smart(
        &info.sender, // This is the CW20 contract address
        &Cw20QueryMsg::TokenInfo {},
    )?;
    
    // UNSAFE: symbol/name can be duplicated
    if token_info.symbol == "USDC" {
        // This could be a fake USDC contract!
        process_usdc_deposit(deps, msg.amount)?;
    }
    
    Ok(Response::new())
}
```

### Secure CW20 Pattern

**Always use contract addresses for CW20 validation**
```rust
// SECURE CW20 handling
pub fn execute_cw20_receive(
    deps: DepsMut,
    info: MessageInfo,
    env: Env,
    msg: Cw20ReceiveMsg,
) -> Result<Response, ContractError> {
    // Check the actual contract address, not the symbol
    let config = CONFIG.load(deps.storage)?;
    
    match info.sender {
        addr if addr == config.usdc_token_address => {
            process_usdc_deposit(deps, msg.amount)?;
        }
        addr if addr == config.approved_tokens.contains(&addr) => {
            process_approved_token_deposit(deps, addr, msg.amount)?;
        }
        _ => return Err(ContractError::UnauthorizedToken {}),
    }
    
    Ok(Response::new())
}
```

### Attack Scenarios

**Fake CW20 Attack**
1. Attacker deploys malicious CW20 contract with symbol "USDC"
2. Contract appears legitimate when queried for token info
3. Victim contract accepts it as real USDC based on symbol
4. Attacker can mint unlimited fake tokens

**Example of vulnerable code:**
```rust
// NEVER DO THIS!
pub fn is_valid_usdc(deps: Deps, token_addr: &Addr) -> Result<bool, ContractError> {
    let token_info: TokenInfoResponse = deps.querier.query_wasm_smart(
        token_addr,
        &Cw20QueryMsg::TokenInfo {},
    )?;
    
    // DANGEROUS: Any contract can claim to be USDC
    Ok(token_info.symbol == "USDC" && token_info.name == "USD Coin")
}
```

### Multi-Token Contract Security

**Secure multi-token handling:**
```rust
#[cw_serde]
pub struct Config {
    pub accepted_tokens: Vec<TokenConfig>,
}

#[cw_serde]
pub struct TokenConfig {
    pub address: Addr,
    pub symbol: String, // For display only
    pub decimals: u8,
    pub min_deposit: Uint128,
}

pub fn execute_receive(
    deps: DepsMut,
    info: MessageInfo,
    env: Env,
    msg: Cw20ReceiveMsg,
) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    
    // Find token by CONTRACT ADDRESS, not symbol
    let token_config = config.accepted_tokens
        .iter()
        .find(|t| t.address == info.sender)
        .ok_or(ContractError::TokenNotAccepted {})?;
    
    // Validate minimum deposit
    if msg.amount < token_config.min_deposit {
        return Err(ContractError::InsufficientDeposit {});
    }
    
    process_token_deposit(deps, &token_config, msg.amount)?;
    Ok(Response::new())
}
```

### Best Practices

**For Native Tokens:**
```rust
// Safe native token validation
pub fn validate_native_payment(
    funds: &[Coin],
    expected_denom: &str,
    min_amount: Uint128,
) -> Result<Uint128, ContractError> {
    let payment = funds.iter()
        .find(|coin| coin.denom == expected_denom)
        .ok_or(ContractError::RequiredPaymentMissing {})?;
    
    if payment.amount < min_amount {
        return Err(ContractError::InsufficientPayment {});
    }
    
    Ok(payment.amount)
}
```

**For CW20 Tokens:**
```rust
// Secure CW20 token registry
pub fn add_accepted_token(
    deps: DepsMut,
    info: MessageInfo,
    token_address: String,
) -> Result<Response, ContractError> {
    // Only admin can add tokens
    ensure_admin(deps.as_ref(), &info.sender)?;
    
    let token_addr = deps.api.addr_validate(&token_address)?;
    
    // Verify it's actually a CW20 contract
    let token_info: TokenInfoResponse = deps.querier.query_wasm_smart(
        &token_addr,
        &Cw20QueryMsg::TokenInfo {},
    )?;
    
    let config = TokenConfig {
        address: token_addr,
        symbol: token_info.symbol, // Store for display
        decimals: token_info.decimals,
        min_deposit: Uint128::from(1000u128),
    };
    
    ACCEPTED_TOKENS.save(deps.storage, &config.address, &config)?;
    
    Ok(Response::new())
}
```

### Key Security Rules

1. **Native tokens**: `funds.denom` checking is safe and sufficient
2. **CW20 tokens**: NEVER trust symbol/name, always validate contract address
3. **Whitelist approach**: Maintain a registry of approved token contracts
4. **Admin controls**: Only trusted entities should add new tokens
5. **Verification**: Query token info to ensure it's actually a CW20 contract

The fundamental difference is that native token denoms are globally unique and managed by the chain, while CW20 token symbols are just strings that any contract can claim. Always design your security model around contract addresses for CW20 tokens.
