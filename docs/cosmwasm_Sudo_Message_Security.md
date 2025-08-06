# CosmWasm Security Guide: Sudo Messages and ICQ

*A comprehensive security guide covering sudo messages, interchain queries, and audit considerations in CosmWasm smart contracts*

---

## Table of Contents

1. [CosmWasm Sudo Security Considerations](#cosmwasm-sudo-security-considerations)
2. [Unauthorized Sudo Access Vectors](#unauthorized-sudo-access-vectors)
3. [Interchain Query (ICQ) Security](#interchain-query-icq-security)
4. [Sudo Entrypoint Usage Beyond ICQ](#sudo-entrypoint-usage-beyond-icq)
5. [Comprehensive Audit Checklist](#comprehensive-audit-checklist)

---

## CosmWasm Sudo Security Considerations

When working with sudo messages in CosmWasm, there are several critical security considerations to keep in mind:

### Access Control and Authorization

The most fundamental concern is ensuring only authorized entities can execute sudo messages. The contract should implement robust checks to verify the caller's identity and permissions before processing any sudo operation. This typically involves validating that the message sender matches expected admin addresses or has the required roles.

### State Validation and Consistency

Sudo messages often bypass normal execution flows and can directly modify contract state. It's crucial to validate all state changes to prevent corruption or inconsistencies. This includes checking that new values are within acceptable ranges, maintaining invariants, and ensuring dependent state variables remain synchronized.

### Input Sanitization and Bounds Checking

Since sudo operations may accept parameters that normal users cannot provide, thorough input validation becomes essential. All parameters should be sanitized, checked for proper formatting, and validated against expected ranges or constraints to prevent overflow conditions or malicious inputs.

### Reentrancy Protection

Sudo messages might interact with other contracts or trigger callbacks. Implement proper reentrancy guards to prevent malicious contracts from exploiting recursive calls during sudo execution. This is particularly important if the sudo operation involves token transfers or external contract interactions.

### Privilege Escalation Prevention

Design sudo operations with the principle of least privilege. Each sudo message should only grant the minimum necessary permissions to accomplish its intended function. Avoid creating overly broad sudo operations that could be misused if compromised.

### Audit Trail and Logging

Implement comprehensive logging for all sudo operations. This includes recording who executed the operation, what changes were made, and when they occurred. This audit trail is crucial for debugging issues and detecting potential misuse.

### Emergency Safeguards

Consider implementing emergency stop mechanisms or time delays for critical sudo operations. This provides a window for detecting and preventing malicious actions before they take effect.

### Code Review and Testing

Sudo functionality should receive extra scrutiny during code reviews since it operates with elevated privileges. Implement comprehensive test suites that specifically target edge cases and potential attack vectors in sudo message handling.

### Migration and Upgrade Considerations

If sudo messages are used for contract migrations or upgrades, ensure proper validation of new code and state migration logic. Consider implementing multi-signature requirements or governance mechanisms for critical upgrade operations.

The key principle is treating sudo messages as high-risk operations that require multiple layers of protection, even though they may seem like convenient administrative tools.

---

## Unauthorized Sudo Access Vectors

There are several ways an unauthorized address could potentially call a sudo message in CosmWasm if proper access controls aren't implemented. Here are the main attack vectors that require careful validation:

### 1. Missing Access Control Checks

The most common vulnerability is simply **not checking the caller at all**:

```rust
// VULNERABLE - No access control
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        SudoMsg::UpdateConfig { new_owner } => {
            // Anyone can call this!
            CONFIG.save(deps.storage, &Config { owner: new_owner })?;
            Ok(Response::new())
        }
    }
}
```

### 2. Incorrect Caller Validation Logic

Even when checks exist, they might be implemented incorrectly:

```rust
// VULNERABLE - Wrong comparison logic
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    
    // This checks if caller is NOT the owner (reversed logic!)
    if env.contract.address != config.owner {
        return Err(ContractError::Unauthorized {});
    }
    // Process sudo message...
}
```

### 3. Using Wrong Address for Validation

```rust
// VULNERABLE - Checking contract address instead of message sender
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    
    // env.contract.address is the contract's own address, not the caller!
    if env.contract.address != config.owner {
        return Err(ContractError::Unauthorized {});
    }
}
```

### 4. Race Conditions in Owner Updates

```rust
// VULNERABLE - Race condition
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        SudoMsg::TransferOwnership { new_owner } => {
            // No validation - anyone who calls this first wins!
            CONFIG.save(deps.storage, &Config { owner: new_owner })?;
            Ok(Response::new())
        }
    }
}
```

### 5. Relying on External Contract Validation

```rust
// VULNERABLE - Trusting external contract without verification
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    // Assuming some external contract validates permissions
    // but not verifying the caller's actual authority
    match msg {
        SudoMsg::CriticalOperation { .. } => {
            // No direct validation here - dangerous!
            execute_critical_operation(deps)?;
            Ok(Response::new())
        }
    }
}
```

### 6. Initialization Vulnerabilities

```rust
// VULNERABLE - No owner set during instantiation
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    _msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    // Contract deployed without setting an owner!
    CONFIG.save(deps.storage, &Config { owner: Addr::unchecked("") })?;
    Ok(Response::new())
}
```

### 7. Logic Bypass Through Message Crafting

```rust
// VULNERABLE - Insufficient message validation
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        SudoMsg::AdminOperation { admin_key, .. } => {
            // Weak validation that could be bypassed
            if admin_key == "secret123" {
                execute_admin_operation(deps)?;
            }
        }
    }
}
```

### Important Note About CosmWasm Sudo

In CosmWasm, **sudo messages are actually called by the blockchain module itself**, not directly by external addresses. The real vulnerability typically occurs in:

1. **The contract logic that triggers sudo calls** - ensuring only authorized contracts/modules can trigger these
2. **Cross-contract calls** that eventually lead to sudo execution
3. **Migration scenarios** where sudo might be called during contract upgrades

### Proper Protection Pattern

```rust
// SECURE - Proper sudo validation pattern
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn sudo(deps: DepsMut, _env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    // In true sudo context, validation often happens at the module level
    // But we can still validate internal state consistency
    
    match msg {
        SudoMsg::EmergencyPause => {
            let mut config = CONFIG.load(deps.storage)?;
            config.paused = true;
            CONFIG.save(deps.storage, &config)?;
            
            Ok(Response::new()
                .add_attribute("action", "emergency_pause")
                .add_attribute("timestamp", env.block.time.to_string()))
        }
    }
}

// The real authorization should happen in the calling context
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::EmergencyPause => {
            let config = CONFIG.load(deps.storage)?;
            
            // THIS is where we validate the caller
            if info.sender != config.owner && !config.emergency_contacts.contains(&info.sender) {
                return Err(ContractError::Unauthorized {});
            }
            
            // Then we might call sudo or perform the operation directly
            pause_contract(deps)?;
            Ok(Response::new())
        }
    }
}
```

The key insight is that while sudo messages themselves might not have traditional "senders" to validate, the **entry points that lead to sudo calls** must have robust authorization checks.

---

## Interchain Query (ICQ) Security

### Can Unauthorized Contracts Call ICQ?

**The short answer is: No, a contract cannot successfully execute ICQ operations on another chain without proper registration and authorization mechanisms.**

### ICQ Registration Requirements

A smart contract must register interchain queries through a formal process that includes:

- **Query Registration**: Explicit registration with specific parameters including query payload and update period
- **Financial Deposits**: Deposit requirements for query creation to prevent spam
- **IBC Connection**: Proper IBC connection between chains with relayer infrastructure

### Technical Barriers Preventing Unauthorized Access

#### 1. IBC Connection Requirements
For Interchain Queries to function, two blockchains need to be interconnected through IBC. Additionally, you need a relayer to send messages between these zones, a module to create and verify the queries, and a system to decipher the rules and enable them for use.

#### 2. Relayer Infrastructure
The interchain query requires an additional ICQ relayer for each IBC connection. ICQ relayer is responsible for relaying answers for each query registered.

#### 3. Registration Process
Even if a contract knows data values, it cannot bypass the formal registration process because:

- **Query Registration**: The ICQ system requires explicit query registration with specific parameters
- **Deposit Requirements**: Financial deposits are required for query creation
- **Module-Level Validation**: The ICQ module validates registered queries before processing

### Potential Attack Scenarios (and Why They Fail)

#### Scenario 1: Direct Data Access Attempt
```rust
// This WILL NOT WORK - no direct cross-chain access
pub fn try_unauthorized_query(deps: DepsMut) -> Result<Response, ContractError> {
    // Cannot directly query another chain's storage
    // Even knowing the key/value structure doesn't help
    let remote_data = some_other_chain_storage.load("key")?; // Impossible!
    Ok(Response::new())
}
```

#### Scenario 2: Forged ICQ Messages
An attacker might try to craft messages that look like ICQ responses, but this fails because:
- ICQ responses go through IBC verification
- Cryptographic proofs are required
- The relayer infrastructure validates message authenticity

#### Scenario 3: Exploiting Existing Registrations
Even if a legitimate ICQ registration exists, unauthorized contracts cannot:
- Intercept the query results (they're routed to the registered contract)
- Modify the query parameters without proper authorization
- Access query results meant for other contracts

### Security Best Practices for ICQ

#### Cross-Contract Authorization Checks
```rust
// SECURE - Verify ICQ caller authorization
pub fn handle_icq_response(
    deps: DepsMut,
    env: Env,
    msg: IcqResponseMsg,
) -> Result<Response, ContractError> {
    // Verify this contract is authorized to receive this ICQ response
    let registered_queries = REGISTERED_QUERIES.load(deps.storage)?;
    
    if !registered_queries.contains(&msg.query_id) {
        return Err(ContractError::UnauthorizedIcqResponse {
            query_id: msg.query_id
        });
    }
    
    // Process the response...
}
```

#### Registration Validation
```rust
// SECURE - Validate ICQ registration
pub fn register_icq(
    deps: DepsMut,
    info: MessageInfo,
    query_params: IcqParams,
) -> Result<Response, ContractError> {
    // Check caller authorization
    let config = CONFIG.load(deps.storage)?;
    if info.sender != config.admin {
        return Err(ContractError::Unauthorized {});
    }
    
    // Validate query parameters
    validate_query_params(&query_params)?;
    
    // Register with proper deposit
    let register_msg = NeutronMsg::RegisterInterchainQuery {
        query_type: query_params.query_type,
        keys: query_params.keys,
        transactions_filter: query_params.filter,
        connection_id: query_params.connection_id,
        update_period: query_params.update_period,
    };
    
    Ok(Response::new().add_message(register_msg))
}
```

---

## Sudo Entrypoint Usage Beyond ICQ

The sudo entrypoint is **not** only invoked on ICQ messages. The sudo entrypoint in CosmWasm is used for various blockchain module-initiated calls, with ICQ being just one use case.

### When Sudo is Invoked

The sudo entrypoint can be called by various Cosmos SDK modules and chain-level operations:

#### 1. Interchain Queries (ICQ) Responses
```rust
#[cw_serde]
pub enum SudoMsg {
    KVQueryResult { query_id: u64 },
    TxQueryResult { query_id: u64, height: u64, data: Binary },
}
```

#### 2. IBC Operations
- IBC packet timeouts
- IBC acknowledgments
- Channel closing operations
- Connection handshake failures

#### 3. Chain Governance Actions
- Emergency chain halts
- Parameter updates
- Module-level administrative operations

#### 4. Scheduled Operations
- Cron jobs or scheduled tasks (on chains that support this)
- Automated maintenance operations

#### 5. Chain Upgrades
- Contract migration during chain upgrades
- State synchronization operations

#### 6. Custom Module Interactions
- Any custom Cosmos SDK module can call sudo on contracts
- Oracle price feeds
- Validator set changes
- Slashing events

### ICQ-Specific Sudo Pattern

For ICQ specifically, the sudo call typically looks like:

```rust
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        // ICQ Key-Value query response
        SudoMsg::KVQueryResult { query_id } => {
            let registered_query = get_registered_kv_query(deps.storage, query_id)?;
            let query_result = get_query_result(deps.storage, query_id)?;
            
            handle_kv_query_result(deps, query_result, registered_query)
        },
        
        // ICQ Transaction query response  
        SudoMsg::TxQueryResult { query_id, height, data } => {
            let registered_query = get_registered_tx_query(deps.storage, query_id)?;
            
            handle_tx_query_result(deps, query_id, height, data, registered_query)
        },
        
        // Other non-ICQ sudo operations
        SudoMsg::EmergencyPause => {
            handle_emergency_pause(deps)
        },
        
        SudoMsg::ChainUpgrade { new_version } => {
            handle_chain_upgrade(deps, new_version)
        }
    }
}
```

### Security Implications

This broader use of sudo creates additional security considerations:

#### Message Type Validation
```rust
// SECURE - Validate sudo message types
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    match msg {
        SudoMsg::KVQueryResult { query_id } => {
            // Verify this query_id was registered by this contract
            if !is_registered_query(deps.storage, query_id)? {
                return Err(ContractError::UnauthorizedQuery { query_id });
            }
            handle_icq_response(deps, query_id)
        },
        
        // Reject unexpected sudo message types
        _ => Err(ContractError::UnsupportedSudoMessage {})
    }
}
```

#### State Consistency Across Sudo Types
```rust
// SECURE - Maintain state consistency across sudo operations
pub fn sudo(deps: DepsMut, env: Env, msg: SudoMsg) -> Result<Response, ContractError> {
    let mut state = CONTRACT_STATE.load(deps.storage)?;
    
    match msg {
        SudoMsg::KVQueryResult { query_id } => {
            // ICQ might update price data
            state.last_price_update = env.block.time;
            update_price_from_icq(deps, &mut state, query_id)?;
        },
        
        SudoMsg::EmergencyPause => {
            // Emergency pause affects all operations
            state.paused = true;
            state.pause_reason = "Chain emergency".to_string();
        }
    }
    
    CONTRACT_STATE.save(deps.storage, &state)?;
    Ok(Response::new())
}
```

---

## Comprehensive Audit Checklist

### 1. Access Control & Authorization

#### 1.1 Caller Verification
- [ ] **Admin/Owner verification**: Verify that only authorized addresses can call sudo functions
- [ ] **Role-based access**: Check if proper role-based permissions are implemented
- [ ] **Multi-signature requirements**: Confirm multi-sig is required for critical operations (if applicable)
- [ ] **Time-locked operations**: Verify time delays are implemented for sensitive operations
- [ ] **Governance integration**: Check if governance mechanisms control sudo access

#### 1.2 Permission Boundaries
- [ ] **Least privilege principle**: Each sudo operation only grants minimum necessary permissions
- [ ] **Operation scoping**: Sudo functions are limited to their intended scope
- [ ] **Privilege escalation prevention**: No paths for unauthorized privilege escalation
- [ ] **Emergency revocation**: Ability to revoke sudo privileges exists

### 2. Input Validation & Sanitization

#### 2.1 Parameter Validation
- [ ] **Type validation**: All input parameters are properly typed and validated
- [ ] **Range checking**: Numeric inputs are within acceptable bounds
- [ ] **String sanitization**: String inputs are properly sanitized and length-limited
- [ ] **Address validation**: All address inputs are validated as proper addresses
- [ ] **Option/enum validation**: All optional and enum values are properly validated

#### 2.2 Business Logic Validation
- [ ] **State consistency**: Operations maintain contract invariants
- [ ] **Dependency validation**: Required dependencies are checked before execution
- [ ] **Precondition checks**: All necessary preconditions are verified
- [ ] **Amount validation**: Token amounts are positive and within limits

### 3. State Management & Data Integrity

#### 3.1 State Modifications
- [ ] **Atomic operations**: State changes are atomic and consistent
- [ ] **State validation**: New state values are validated before storage
- [ ] **Rollback mechanisms**: Failed operations properly rollback state changes
- [ ] **State synchronization**: Related state variables remain synchronized

#### 3.2 Storage Security
- [ ] **Storage key validation**: Storage keys are properly validated
- [ ] **Data serialization**: Proper serialization/deserialization of complex data
- [ ] **Storage limits**: Storage operations respect size and quantity limits
- [ ] **Orphaned data cleanup**: Unused storage is properly cleaned up

### 4. Reentrancy & External Interactions

#### 4.1 Reentrancy Protection
- [ ] **Reentrancy guards**: Proper guards against recursive calls
- [ ] **State locking**: Critical sections are protected from concurrent access
- [ ] **External call ordering**: External calls are made after state updates
- [ ] **Callback validation**: Callbacks from external contracts are properly validated

#### 4.2 Cross-Contract Interactions
- [ ] **Contract address validation**: External contract addresses are validated
- [ ] **Message construction**: Outgoing messages are properly constructed
- [ ] **Error handling**: Failures in external calls are properly handled
- [ ] **Gas limit considerations**: Sufficient gas is reserved for external operations

### 5. Error Handling & Recovery

#### 5.1 Error Management
- [ ] **Comprehensive error handling**: All error cases are handled
- [ ] **Informative error messages**: Error messages are clear but not revealing sensitive info
- [ ] **Error propagation**: Errors are properly propagated up the call stack
- [ ] **Panic prevention**: Operations that could panic are protected

#### 5.2 Recovery Mechanisms
- [ ] **Emergency stop**: Emergency pause/stop functionality exists
- [ ] **Recovery procedures**: Clear recovery procedures for failed operations
- [ ] **State restoration**: Ability to restore contract to previous valid state
- [ ] **Manual intervention**: Admin can manually intervene when needed

### 6. Logging & Auditability

#### 6.1 Event Logging
- [ ] **Operation logging**: All sudo operations are logged with events
- [ ] **Parameter logging**: Important parameters are included in events
- [ ] **Caller identification**: Events include caller information
- [ ] **Timestamp recording**: Operations include timestamp information

#### 6.2 Audit Trail
- [ ] **Comprehensive audit trail**: Complete history of sudo operations
- [ ] **Data retention**: Audit data is retained appropriately
- [ ] **Query capabilities**: Audit data can be queried effectively
- [ ] **Tamper evidence**: Audit trail cannot be modified after creation

### 7. Testing & Quality Assurance

#### 7.1 Test Coverage
- [ ] **Unit test coverage**: All sudo functions have comprehensive unit tests
- [ ] **Integration testing**: Sudo operations tested in realistic scenarios
- [ ] **Edge case testing**: Boundary conditions and edge cases are tested
- [ ] **Negative testing**: Error conditions and invalid inputs are tested

#### 7.2 Security Testing
- [ ] **Fuzzing**: Sudo functions tested with random/malicious inputs
- [ ] **Load testing**: Performance under high load is tested
- [ ] **Stress testing**: Behavior under resource constraints is verified
- [ ] **Attack simulation**: Common attack vectors are simulated and defended against

### 8. Documentation & Code Quality

#### 8.1 Code Documentation
- [ ] **Function documentation**: All sudo functions are well-documented
- [ ] **Security considerations**: Security implications are documented
- [ ] **Usage examples**: Clear examples of proper usage
- [ ] **Risk assessment**: Known risks and mitigation strategies documented

#### 8.2 Code Quality
- [ ] **Code clarity**: Sudo code is clear and readable
- [ ] **Complexity management**: Complex operations are broken down appropriately
- [ ] **Consistent patterns**: Consistent coding patterns throughout
- [ ] **Code reviews**: Multiple reviewers have examined the code

### 9. Migration & Upgrade Safety

#### 9.1 Contract Migrations
- [ ] **Migration validation**: New contract versions are properly validated
- [ ] **State migration**: State migration is handled correctly
- [ ] **Backward compatibility**: Breaking changes are properly managed
- [ ] **Migration testing**: Migration procedures are thoroughly tested

#### 9.2 Upgrade Mechanisms
- [ ] **Controlled upgrades**: Upgrades require proper authorization
- [ ] **Version compatibility**: Version compatibility is maintained
- [ ] **Rollback capability**: Ability to rollback failed upgrades
- [ ] **Communication**: Upgrade plans are communicated to stakeholders

### 10. Specific CosmWasm Considerations

#### 10.1 CosmWasm Specifics
- [ ] **Gas optimization**: Sudo operations are gas-efficient
- [ ] **Query consistency**: Queries return consistent data with sudo modifications
- [ ] **Module integration**: Proper integration with Cosmos SDK modules
- [ ] **IBC compatibility**: IBC operations (if any) are properly handled

#### 10.2 Chain Integration
- [ ] **Chain-specific features**: Chain-specific functionality is properly utilized
- [ ] **Governance compatibility**: Compatible with chain governance mechanisms
- [ ] **Validator considerations**: Impact on validators is considered
- [ ] **Network effects**: Network-wide effects are evaluated

### 11. ICQ-Specific Security

#### 11.1 ICQ Registration
- [ ] **Query registration validation**: ICQ queries are properly registered with deposits
- [ ] **Query parameter validation**: All ICQ parameters are validated before registration
- [ ] **Authorization checks**: Only authorized entities can register ICQ queries
- [ ] **Query ID tracking**: All registered query IDs are properly tracked

#### 11.2 ICQ Response Handling
- [ ] **Response authorization**: ICQ responses are validated against registered queries
- [ ] **Data validation**: ICQ response data is properly validated before use
- [ ] **Timeout handling**: ICQ timeouts are properly handled
- [ ] **Error recovery**: ICQ errors have appropriate recovery mechanisms

### 12. Multi-Sudo-Type Validation

#### 12.1 Message Type Security
- [ ] **Sudo message type validation**: Verify only expected sudo message types are handled
- [ ] **Cross-sudo-type state management**: Ensure different sudo operations don't create state conflicts
- [ ] **Sudo message authenticity**: Verify sudo messages come from legitimate blockchain modules
- [ ] **Resource limits**: Ensure sudo operations respect gas and storage limits
- [ ] **Error isolation**: Failures in one sudo operation don't affect others

## Critical Red Flags ⚠️

Stop audit immediately and escalate if any of these are found:
- [ ] Sudo functions with no access control
- [ ] Direct state manipulation without validation
- [ ] Hardcoded private keys or secrets
- [ ] Unrestricted external contract calls
- [ ] Missing error handling in critical paths
- [ ] Reentrancy vulnerabilities in fund transfers
- [ ] Privilege escalation attack vectors
- [ ] Missing emergency stop mechanisms for critical functions

## Audit Completion

### Final Verification
- [ ] **All checklist items reviewed**: Every item has been examined
- [ ] **Issues documented**: All found issues are properly documented
- [ ] **Risk assessment complete**: Overall risk level has been assessed
- [ ] **Recommendations provided**: Clear recommendations for improvements
- [ ] **Sign-off obtained**: Appropriate stakeholders have signed off

### Post-Audit
- [ ] **Issue tracking**: System in place to track issue resolution
- [ ] **Follow-up planned**: Follow-up audits are scheduled if needed
- [ ] **Deployment checklist**: Pre-deployment security checklist completed
- [ ] **Monitoring setup**: Post-deployment monitoring is configured

---

## Key Takeaways

1. **Sudo messages are high-privilege operations** that require multiple layers of security protection
2. **ICQ operations have built-in security mechanisms** through registration requirements and IBC validation
3. **Sudo entrypoints serve multiple purposes** beyond just ICQ, including IBC operations, governance actions, and chain upgrades
4. **Authorization checks are critical** at all levels of sudo operation handling
5. **Comprehensive testing and auditing** are essential for sudo functionality due to elevated privileges
6. **State consistency** must be maintained across different types of sudo operations

---

**Document Version:** 1.0  
**Last Updated:** August 2025  
**Scope:** CosmWasm Sudo Messages, ICQ Security, Audit Guidelines
