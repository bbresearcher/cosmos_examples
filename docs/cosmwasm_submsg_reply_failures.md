# CosmWasm Submessage Reply Processing Failures

## Overview

In CosmWasm, when a submessage receives a reply but fails during reply processing, the behavior depends on how the submessage was configured and what type of failure occurs.

## Reply Processing Failures

When a submessage gets a successful reply but fails while processing that reply in your contract's `reply` entry point:

### If the submessage was sent with `ReplyOn::Always` or `ReplyOn::Success`:
- The reply handler executes but encounters an error (panic, contract error, etc.)
- The entire transaction gets reverted, including the original submessage execution
- Even though the submessage itself succeeded, the failure in reply processing causes a rollback
- Any state changes made by both the submessage and your contract are undone

### If the submessage was sent with `ReplyOn::Error`:
- This scenario shouldn't occur since you only get replies on errors, not successes
- If somehow you get a success reply when expecting only error replies, it would likely cause unexpected behavior

## Common Failure Scenarios in Reply Processing

1. **Contract Logic Errors**: Your reply handler has a bug or invalid logic
2. **State Access Issues**: Trying to read/write state that doesn't exist or is corrupted
3. **Insufficient Gas**: Running out of gas during reply processing
4. **Serialization/Deserialization Errors**: Failed parsing of the reply data

## Best Practices

To handle this robustly:

```rust
pub fn reply(deps: DepsMut, env: Env, msg: Reply) -> Result<Response, ContractError> {
    match msg.result {
        SubMsgResult::Ok(res) => {
            // Handle successful submessage
            // Use proper error handling here
            process_success_reply(deps, env, res)
                .map_err(|e| ContractError::ReplyProcessingError(e.to_string()))
        }
        SubMsgResult::Err(err) => {
            // Handle failed submessage
            Ok(Response::new().add_attribute("submsg_error", err))
        }
    }
}
```

## Key Takeaway

The key point is that reply processing failures cause transaction rollback, so the apparent "success" of the submessage gets undone. This ensures transaction atomicity but requires careful error handling in your reply logic.
