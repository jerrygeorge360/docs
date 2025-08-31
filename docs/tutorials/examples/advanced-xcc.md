---
id: advanced cross-contract
title: Understanding Cross-Contract Calls in NEAR
description: "Master the fundamentals of cross-contract interactions in NEAR Protocol, including asynchronous patterns, Promises, and callback handling."
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';
import {CodeTabs, Language, Github} from "@site/src/components/codetabs"

Cross-contract calls in NEAR work differently from most blockchains due to NEAR's sharded architecture. This fundamental difference enables powerful patterns that would be impossible in synchronous systems, but requires understanding some key concepts first.

:::info Advanced Examples

For complex multi-contract patterns and complete working examples, see [Complex Cross Contract Calls](advanced-xcc.md)

:::

---

## Why NEAR is Different: The Asynchronous Advantage

Unlike Ethereum and other blockchains where contract calls happen synchronously within a single transaction, NEAR operates asynchronously due to its sharded nature. This isn't a limitation—it's a design choice that enables massive scalability.

Understanding these two principles is crucial:

- **Cross-contract calls are independent**: Each contract runs in its own execution environment
- **Cross-contract calls are asynchronous**: You cannot get immediate results from external calls

Think of it like sending emails instead of making phone calls. You send your message, continue with other work, and handle the response when it arrives.

---

## Core Patterns in Cross-Contract Calls

NEAR's asynchronous design enables three distinct interaction patterns:

### 1. Sequential Actions (Batching)

Execute multiple methods on the same contract in order. If any method fails, the entire batch reverts:

<CodeTabs>
  <Language value="js" language="js">

```javascript
// Batch multiple calls to maintain atomicity
Promise.create_batch("target.testnet")
  .function_call("first_method", {}, 0, 10**12)
  .function_call("second_method", {}, 0, 10**12);
```

  </Language>
  <Language value="rust" language="rust">

```rust
// Batch multiple calls to maintain atomicity
Promise::new("target.testnet".parse().unwrap())
  .function_call("first_method".to_string(), vec![], 0, 10_u64.pow(12))
  .and(Promise::new("target.testnet".parse().unwrap())
    .function_call("second_method".to_string(), vec![], 0, 10_u64.pow(12)))
```

  </Language>
</CodeTabs>

### 2. Parallel Execution

Call multiple contracts simultaneously. Failures in one contract don't affect others:

<CodeTabs>
  <Language value="js" language="js">

```javascript
// Execute multiple contracts in parallel
Promise.all([
    Promise.create("contract-a.testnet").function_call("foo", {}, 0, 10**12),
    Promise.create("contract-b.testnet").function_call("bar", {}, 0, 10**12)
]);
```

  </Language>
  <Language value="rust" language="rust">

```rust
// Execute multiple contracts in parallel
let promise_a = Promise::new("contract-a.testnet".parse().unwrap())
    .function_call("foo".to_string(), vec![], 0, 10_u64.pow(12));
let promise_b = Promise::new("contract-b.testnet".parse().unwrap())
    .function_call("bar".to_string(), vec![], 0, 10_u64.pow(12));
```

  </Language>
</CodeTabs>

### 3. Callback Handling

Retrieve and process responses after external calls complete:

<CodeTabs>
  <Language value="js" language="js">

```javascript
// Chain calls with callbacks
CrossContract("external.testnet")
  .call("get_data", { param: "value" })
  .then("process_response", { context: "additional_data" })
  .value();
```

  </Language>
  <Language value="rust" language="rust">

```rust
// Chain calls with callbacks
ext_contract::ext("external.testnet".parse().unwrap())
  .get_data("value".to_string())
  .then(Self::ext(env::current_account_id())
    .process_response("additional_data".to_string()))
```

  </Language>
</CodeTabs>

---

## Understanding Promises

A Promise in NEAR represents a scheduled instruction for the blockchain—an asynchronous action that executes after the current transaction succeeds.

### Basic Promise Actions

Promises can perform several types of actions:

- **Function calls**: Execute methods on other contracts
- **Token transfers**: Send NEAR tokens
- **Batch creation**: Group multiple actions together

<CodeTabs>
  <Language value="js" language="js">

```javascript
import { Promise } from 'near-sdk-js';

// Schedule a function call
Promise.create("other_contract.testnet")
  .function_call("do_something", {}, 0, 10**12);

// This schedules the call but doesn't execute it immediately
```

  </Language>
  <Language value="rust" language="rust">

```rust
use near_sdk::Promise;

// Schedule a function call
Promise::new("other_contract.testnet".parse().unwrap())
  .function_call("do_something".to_string(), vec![], 0, 10_u64.pow(12));

// This schedules the call but doesn't execute it immediately
```

  </Language>
</CodeTabs>

---

## Creating Cross-Contract Calls with Callbacks

To create meaningful cross-contract interactions, you'll typically want to process the results. Here's how to set up calls with callbacks:

### High-Level API (Recommended)

<CodeTabs>
  <Language value="js" language="js">

```javascript
import { near, call, view, NearBindgen, ONE_TGAS } from 'near-sdk-js';

@NearBindgen({})
export class CrossContractExample {
  
  @call({})
  fetch_greeting({ target_contract }) {
    // Clean and readable approach
    return CrossContract(target_contract).call(
      "get_greeting",
      { name: "World" }
    ).then(
      "greeting_callback",
      { timestamp: near.blockTimestamp() }
    ).value();
  }
}
```

  </Language>
  <Language value="rust" language="rust">

```rust
use near_sdk::{near_bindgen, Promise, env, Gas};

#[near_bindgen]
impl CrossContractExample {
    
    pub fn fetch_greeting(&mut self, target_contract: AccountId) -> Promise {
        ext_contract::ext(target_contract)
            .get_greeting("World".to_string())
            .then(Self::ext(env::current_account_id())
                .greeting_callback(env::block_timestamp()))
    }
}
```

  </Language>
</CodeTabs>

### Low-Level Promise API

For fine-grained control, use the Promise API directly:

<CodeTabs>
  <Language value="js" language="js">

```javascript
import { Promise, near, ONE_TGAS } from 'near-sdk-js';

// Detailed Promise construction
Promise.create("external_contract.testnet")
  .function_call(
    "get_greeting",
    { name: "World" },
    0,                    // Deposit in yoctoNEAR
    5 * ONE_TGAS         // Gas allowance
  )
  .then(near.currentAccountId())
  .function_call(
    "greeting_callback",
    { timestamp: near.blockTimestamp() }
  )
  .value();
```

  </Language>
  <Language value="rust" language="rust">

```rust
use near_sdk::{Promise, env, Gas};

// Detailed Promise construction
Promise::new("external_contract.testnet".parse().unwrap())
    .function_call(
        "get_greeting".to_string(),
        json!({"name": "World"}).to_string().into_bytes(),
        0,                      // Deposit in yoctoNEAR  
        Gas(5_000_000_000_000)  // Gas allowance
    )
    .then(Promise::new(env::current_account_id())
        .function_call(
            "greeting_callback".to_string(),
            json!({"timestamp": env::block_timestamp()}).to_string().into_bytes(),
            0,
            Gas(5_000_000_000_000)
        ))
```

  </Language>
</CodeTabs>

---

## Implementing Callbacks

Callbacks are where your cross-contract calls complete their journey. Here's how to handle them properly:

<CodeTabs>
  <Language value="js" language="js">

```javascript
import { near, call, NearBindgen } from 'near-sdk-js';

@NearBindgen({})
export class CrossContractExample {
  
  @call({ privateFunction: true })
  greeting_callback({ result, timestamp }) {
    // Access the promise result
    const promiseResult = near.promiseResult(0);
    
    if (promiseResult.length === 0) {
      // External call failed
      return {
        success: false,
        message: "Failed to get greeting",
        timestamp
      };
    }
    
    // Parse successful result
    const greeting = JSON.parse(promiseResult);
    return {
      success: true,
      greeting,
      message: `Successfully received: ${greeting}`,
      timestamp
    };
  }
}
```

  </Language>
  <Language value="rust" language="rust">

```rust
use near_sdk::{near_bindgen, env, PromiseResult, serde_json};

#[near_bindgen]
impl CrossContractExample {
    
    #[private]
    pub fn greeting_callback(&mut self, timestamp: u64) -> serde_json::Value {
        // Check the promise result
        match env::promise_result(0) {
            PromiseResult::NotReady => unreachable!(),
            PromiseResult::Failed => {
                // External call failed
                json!({
                    "success": false,
                    "message": "Failed to get greeting",
                    "timestamp": timestamp
                })
            }
            PromiseResult::Successful(data) => {
                // Parse successful result
                let greeting: String = serde_json::from_slice(&data).unwrap();
                json!({
                    "success": true,
                    "greeting": greeting,
                    "message": format!("Successfully received: {}", greeting),
                    "timestamp": timestamp
                })
            }
        }
    }
}
```

  </Language>
</CodeTabs>

:::warning Callback Execution
Your callback will execute whether the external contract succeeds or fails. Always check the promise result status and handle failures appropriately.
:::

---

## Critical Considerations for Production

When building production applications with cross-contract calls, keep these important points in mind:

### Manual State Rollbacks

If an external function fails, your callback executes, but any state changes made in the original call won't automatically revert. You must handle cleanup manually:

<CodeTabs>
  <Language value="js" language="js">

```javascript
@call({ privateFunction: true })
payment_callback({ user_id, amount, original_balance }) {
  const result = near.promiseResult(0);
  
  if (result.length === 0) {
    // External payment failed - restore user's balance
    this.user_balances.set(user_id, original_balance);
    this.refund_user(user_id, amount);
    return { success: false, message: "Payment failed" };
  }
  
  // Payment succeeded - process normally
  return { success: true, message: "Payment completed" };
}
```

  </Language>
  <Language value="rust" language="rust">

```rust
#[private]
pub fn payment_callback(&mut self, user_id: AccountId, amount: u128, original_balance: u128) {
    match env::promise_result(0) {
        PromiseResult::Failed => {
            // External payment failed - restore user's balance
            self.user_balances.insert(&user_id, &original_balance);
            self.refund_user(user_id, amount);
        }
        PromiseResult::Successful(_) => {
            // Payment succeeded - process normally
            near_sdk::log!("Payment completed successfully");
        }
        PromiseResult::NotReady => unreachable!(),
    }
}
```

  </Language>
</CodeTabs>

### Gas and Token Management

When your contract attaches NEAR tokens to a cross-contract call that fails, those funds return to your contract—not the original caller. Ensure your callbacks handle refunds appropriately.

### Error Handling Best Practices

Always implement comprehensive error handling in your callbacks:

1. **Check promise results**: Verify success before processing data
2. **Validate returned data**: Don't assume external contracts return expected formats
3. **Implement fallbacks**: Have strategies for when external calls fail
4. **Log failures**: Help with debugging and monitoring

---

## Why This Design is Powerful

NEAR's asynchronous approach might seem more complex initially, but it provides significant advantages:

**Scalability**: Sharded execution means calls don't compete for the same resources
**Flexibility**: Design sophisticated workflows impossible in synchronous systems  
**Reliability**: Failed calls in one contract don't cascade failures to others
**Performance**: Parallel execution enables faster overall transaction processing

---

## Getting Started

Ready to implement cross-contract calls? Here's a recommended progression:

1. **Start with simple calls**: Practice basic Promise creation and callback handling
2. **Experiment with patterns**: Try batching, parallel calls, and callback chains
3. **Handle edge cases**: Implement proper error handling and state management
4. **Test thoroughly**: Use sandbox testing to validate your cross-contract logic

The asynchronous nature of NEAR requires a mental shift from traditional blockchain development, but mastering these patterns opens up possibilities for building sophisticated, scalable applications that leverage NEAR's unique architecture.

:::tip Next Steps
Once you're comfortable with these fundamentals, explore the [Complex Cross Contract Call examples](advanced-xcc.md) to see these patterns in action with complete working code.
:::
