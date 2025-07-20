# CosmWasm Input Validation Security Audit Checklist
(Created in Collaboration with AI, these tips are just best practices, always double check documentation to be sure concepts are correct)

## 1. Message Parameter Validation

### ✅ Required Field Validation
Check that all required fields are present and non-empty.

```rust
// ❌ BAD: No validation
pub fn execute_transfer(amount: Uint128, recipient: String) -> Result<Response, ContractError> {
    // Direct usage without validation
}

// ✅ GOOD: Proper validation
pub fn execute_transfer(amount: Uint128, recipient: String) -> Result<Response, ContractError> {
    if amount.is_zero() {
        return Err(ContractError::InvalidAmount {});
    }
    if recipient.trim().is_empty() {
        return Err(ContractError::InvalidRecipient {});
    }
    // Continue with logic
}
```

### ✅ Address Validation
Always validate addresses before using them.

```rust
// ❌ BAD: Using addresses without validation
pub fn execute_send(recipient: String, amount: Uint128) -> Result<Response, ContractError> {
    // Direct usage - could be invalid address
    let msg = BankMsg::Send {
        to_address: recipient,
        amount: coins(amount.u128(), "token"),
    };
}

// ✅ GOOD: Address validation
pub fn execute_send(recipient: String, amount: Uint128, deps: Deps) -> Result<Response, ContractError> {
    let recipient_addr = deps.api.addr_validate(&recipient)?;
    
    let msg = BankMsg::Send {
        to_address: recipient_addr.to_string(),
        amount: coins(amount.u128(), "token"),
    };
}
```

### ✅ Numerical Range Validation
Validate numerical inputs are within expected ranges.

```rust
// ❌ BAD: No bounds checking
pub fn set_fee_percentage(percentage: u64) -> Result<Response, ContractError> {
    // Could be > 100% or cause overflow
    CONFIG.update(deps.storage, |mut config| {
        config.fee_percentage = percentage;
        Ok(config)
    })?;
}

// ✅ GOOD: Range validation
pub fn set_fee_percentage(percentage: u64) -> Result<Response, ContractError> {
    if percentage > 10000 { // 100.00% in basis points
        return Err(ContractError::InvalidFeePercentage {
            max: 10000,
            received: percentage,
        });
    }
    
    CONFIG.update(deps.storage, |mut config| {
        config.fee_percentage = percentage;
        Ok(config)
    })?;
}
```

## 2. String and Data Validation

### ✅ String Length Limits
Prevent DoS attacks through excessively long strings.

```rust
const MAX_NAME_LENGTH: usize = 50;
const MAX_DESCRIPTION_LENGTH: usize = 500;

// ❌ BAD: No length limits
pub fn create_token(name: String, description: String) -> Result<Response, ContractError> {
    // Could store unlimited data, causing DoS
}

// ✅ GOOD: Length validation
pub fn create_token(name: String, description: String) -> Result<Response, ContractError> {
    if name.len() > MAX_NAME_LENGTH {
        return Err(ContractError::NameTooLong {
            max: MAX_NAME_LENGTH,
            received: name.len(),
        });
    }
    
    if description.len() > MAX_DESCRIPTION_LENGTH {
        return Err(ContractError::DescriptionTooLong {
            max: MAX_DESCRIPTION_LENGTH,
            received: description.len(),
        });
    }
}
```

### ✅ String Content Validation
Validate string contents for allowed characters.

```rust
// ✅ GOOD: Content validation
pub fn set_token_symbol(symbol: String) -> Result<Response, ContractError> {
    if !symbol.chars().all(|c| c.is_ascii_alphanumeric()) {
        return Err(ContractError::InvalidSymbol {
            reason: "Only alphanumeric characters allowed".to_string(),
        });
    }
    
    if symbol.len() < 3 || symbol.len() > 8 {
        return Err(ContractError::InvalidSymbolLength {
            min: 3,
            max: 8,
            received: symbol.len(),
        });
    }
}
```

## 3. Authorization and Permission Validation

### ✅ Admin/Owner Validation
Always verify caller has required permissions.

```rust
// ❌ BAD: No authorization check
pub fn execute_admin_action(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
    // Anyone can call this
    CONFIG.update(deps.storage, |mut config| {
        config.critical_param = new_value;
        Ok(config)
    })?;
}

// ✅ GOOD: Authorization validation
pub fn execute_admin_action(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    if info.sender != config.admin {
        return Err(ContractError::Unauthorized {});
    }
    
    CONFIG.update(deps.storage, |mut config| {
        config.critical_param = new_value;
        Ok(config)
    })?;
}
```

### ✅ Whitelist/Blacklist Validation
Check against allowed/forbidden lists.

```rust
// ✅ GOOD: Whitelist validation
pub fn execute_restricted_action(
    deps: DepsMut,
    info: MessageInfo,
) -> Result<Response, ContractError> {
    let whitelist = WHITELIST.load(deps.storage)?;
    if !whitelist.contains(&info.sender) {
        return Err(ContractError::NotWhitelisted {
            address: info.sender.to_string(),
        });
    }
}
```

## 4. Financial Validation

### ✅ Payment Amount Validation
Validate sent funds match expectations.

```rust
// ❌ BAD: No payment validation
pub fn execute_purchase(deps: DepsMut, info: MessageInfo, item_id: u64) -> Result<Response, ContractError> {
    // Assumes correct payment without checking
}

// ✅ GOOD: Payment validation
pub fn execute_purchase(deps: DepsMut, info: MessageInfo, item_id: u64) -> Result<Response, ContractError> {
    let item = ITEMS.load(deps.storage, item_id)?;
    
    // Validate exact payment
    let payment = must_pay(&info, &item.price_denom)?;
    if payment != item.price_amount {
        return Err(ContractError::IncorrectPayment {
            expected: item.price_amount,
            received: payment,
        });
    }
}
```

### ✅ Balance Validation
Check sufficient balance before transfers.

```rust
// ✅ GOOD: Balance validation
pub fn execute_withdraw(
    deps: DepsMut,
    info: MessageInfo,
    amount: Uint128,
) -> Result<Response, ContractError> {
    let balance = BALANCES.load(deps.storage, &info.sender)?;
    if balance < amount {
        return Err(ContractError::InsufficientBalance {
            available: balance,
            requested: amount,
        });
    }
    
    BALANCES.update(deps.storage, &info.sender, |bal| -> Result<_, ContractError> {
        Ok(bal.unwrap_or_default().checked_sub(amount)?)
    })?;
}
```

## 5. State Validation

### ✅ Contract State Validation
Verify contract is in correct state for operations.

```rust
#[derive(Serialize, Deserialize, Clone, Debug, PartialEq)]
pub enum ContractState {
    Active,
    Paused,
    Migrating,
    Frozen,
}

// ✅ GOOD: State validation
pub fn execute_trade(deps: DepsMut, info: MessageInfo) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    
    match config.state {
        ContractState::Active => {
            // Continue with trade logic
        }
        ContractState::Paused => {
            return Err(ContractError::ContractPaused {});
        }
        ContractState::Frozen => {
            return Err(ContractError::ContractFrozen {});
        }
        ContractState::Migrating => {
            return Err(ContractError::ContractMigrating {});
        }
    }
}
```

### ✅ Time-based Validation
Validate time constraints and deadlines.

```rust
// ✅ GOOD: Time validation
pub fn execute_claim_reward(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
) -> Result<Response, ContractError> {
    let config = CONFIG.load(deps.storage)?;
    
    if env.block.time < config.reward_start_time {
        return Err(ContractError::RewardNotStarted {
            start_time: config.reward_start_time,
            current_time: env.block.time,
        });
    }
    
    if env.block.time > config.reward_end_time {
        return Err(ContractError::RewardExpired {
            end_time: config.reward_end_time,
            current_time: env.block.time,
        });
    }
}
```

## 6. Data Structure Validation

### ✅ Vector/Array Length Validation
Prevent DoS through large arrays.

```rust
const MAX_RECIPIENTS: usize = 100;

// ❌ BAD: No length limits
pub fn execute_batch_transfer(recipients: Vec<(String, Uint128)>) -> Result<Response, ContractError> {
    // Could have thousands of recipients, causing DoS
}

// ✅ GOOD: Array length validation
pub fn execute_batch_transfer(recipients: Vec<(String, Uint128)>) -> Result<Response, ContractError> {
    if recipients.is_empty() {
        return Err(ContractError::EmptyRecipientList {});
    }
    
    if recipients.len() > MAX_RECIPIENTS {
        return Err(ContractError::TooManyRecipients {
            max: MAX_RECIPIENTS,
            received: recipients.len(),
        });
    }
    
    // Validate each recipient
    for (address, amount) in &recipients {
        if amount.is_zero() {
            return Err(ContractError::ZeroAmount {});
        }
        deps.api.addr_validate(address)?;
    }
}
```

### ✅ Unique Key Validation
Prevent duplicate entries where uniqueness is required.

```rust
// ✅ GOOD: Uniqueness validation
pub fn execute_add_validator(
    deps: DepsMut,
    validator_addr: String,
) -> Result<Response, ContractError> {
    let validator = deps.api.addr_validate(&validator_addr)?;
    
    if VALIDATORS.has(deps.storage, &validator) {
        return Err(ContractError::ValidatorAlreadyExists {
            validator: validator.to_string(),
        });
    }
    
    VALIDATORS.save(deps.storage, &validator, &ValidatorInfo::default())?;
}
```

## 7. Overflow and Underflow Prevention
### SHOLUD ALWAYS HAVE `overflow-checks = true` in Cargo.toml file
### ✅ Safe Mathematical Operations
Use checked arithmetic operations.

```rust
// ❌ BAD: Potential overflow
pub fn calculate_reward(base: u64, multiplier: u64) -> u64 {
    base * multiplier // Could overflow
}

// ✅ GOOD: Checked arithmetic
pub fn calculate_reward(base: u64, multiplier: u64) -> Result<u64, ContractError> {
    base.checked_mul(multiplier)
        .ok_or(ContractError::Overflow {
            operation: "reward calculation".to_string(),
        })
}

// ✅ GOOD: Using Uint128 with checked operations
pub fn calculate_fee(amount: Uint128, fee_bps: u64) -> Result<Uint128, ContractError> {
    amount
        .checked_mul(Uint128::from(fee_bps))?
        .checked_div(Uint128::from(10000u64)) // 10000 basis points = 100%
        .map_err(|_| ContractError::DivisionByZero {})
}
```

## 8. Custom Error Types

### ✅ Comprehensive Error Handling
Define specific error types for different validation failures.

```rust
#[derive(Error, Debug, PartialEq)]
pub enum ContractError {
    #[error("{0}")]
    Std(#[from] StdError),

    #[error("Unauthorized")]
    Unauthorized {},

    #[error("Invalid amount: must be greater than zero")]
    InvalidAmount {},

    #[error("Invalid address: {address}")]
    InvalidAddress { address: String },

    #[error("String too long: max {max}, received {received}")]
    StringTooLong { max: usize, received: usize },

    #[error("Value out of range: must be between {min} and {max}, received {value}")]
    ValueOutOfRange { min: u64, max: u64, value: u64 },

    #[error("Insufficient balance: available {available}, requested {requested}")]
    InsufficientBalance { available: Uint128, requested: Uint128 },

    #[error("Contract is paused")]
    ContractPaused {},

    #[error("Too many items in list: max {max}, received {received}")]
    TooManyItems { max: usize, received: usize },

    #[error("Duplicate entry: {key}")]
    DuplicateEntry { key: String },

    #[error("Arithmetic overflow in {operation}")]
    Overflow { operation: String },
}
```

## 9. Input Sanitization Utilities

### ✅ Helper Functions for Common Validations

```rust
pub mod validation {
    use super::*;

    pub fn validate_non_empty_string(value: &str, field_name: &str) -> Result<(), ContractError> {
        if value.trim().is_empty() {
            return Err(ContractError::EmptyField {
                field: field_name.to_string(),
            });
        }
        Ok(())
    }

    pub fn validate_string_length(
        value: &str,
        min: usize,
        max: usize,
        field_name: &str,
    ) -> Result<(), ContractError> {
        let len = value.len();
        if len < min || len > max {
            return Err(ContractError::InvalidStringLength {
                field: field_name.to_string(),
                min,
                max,
                received: len,
            });
        }
        Ok(())
    }

    pub fn validate_percentage(value: u64) -> Result<(), ContractError> {
        if value > 10000 {
            return Err(ContractError::InvalidPercentage {
                max: 10000,
                received: value,
            });
        }
        Ok(())
    }

    pub fn validate_positive_amount(amount: Uint128) -> Result<(), ContractError> {
        if amount.is_zero() {
            return Err(ContractError::ZeroAmount {});
        }
        Ok(())
    }
}
```

## Audit Checklist Summary

When auditing CosmWasm contracts for input validation, ensure:

- [ ] All user inputs are validated before processing
- [ ] Address validation using `deps.api.addr_validate()`
- [ ] Numerical range and overflow checks
- [ ] String length and content validation
- [ ] Authorization and permission checks
- [ ] Payment and balance validation
- [ ] Contract state validation
- [ ] Array/vector length limits
- [ ] Uniqueness constraints where required
- [ ] Proper error handling with descriptive messages
- [ ] Time-based validation for deadlines
- [ ] Safe mathematical operations using checked arithmetic

## Security Best Practices

1. **Fail Fast**: Validate inputs at the beginning of functions
2. **Explicit Validation**: Don't assume inputs are valid
3. **Descriptive Errors**: Provide clear error messages for debugging
4. **Consistent Patterns**: Use similar validation patterns across the contract
5. **Gas Efficiency**: Balance thorough validation with gas costs
6. **Unit Testing**: Write comprehensive tests for all validation scenarios
