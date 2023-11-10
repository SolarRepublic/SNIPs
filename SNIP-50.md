---
snip: 50
title: "Evaporation: Privacy for Wasm Execution Gas"
status: Draft
type: Specification
author: "Blake Regalia (@blake-regalia), Ben Adams (@darwinzer0)"
created: 2023-04-28
---

# SNIP-50 - Evaporation: Privacy for Wasm Execution Gas

This document describes a specification for contracts to implement, as well as a set of best practices to use, in order to prevent leaking execution information through the `gas_used` transaction metadata.

**Evaporation** is a concept that was introduced to overcome privacy risks associated with the publicly viewable `gas_used` field in transaction results' metadata. Evaporation refers to the practice of deliberately consuming extra gas during execution in order to pad the metered `gas_used` amount before it leaves the enclave.

With evaporation, clients can now instruct contracts on exactly how much gas their execution should consume, yielding a consistent `gas_used` across all methods. Evaporation is similar to the `padding` field in this regard; it allows clients to parameterize their message in order to mitigate data leaking through the publicly viewable execution metadata. Whereas `padding` is used to fill the length of encrypted messages, _evaporation_ is used to fill the `gas_used` field.


## Rationale

The public `gas_used` field of a contract execution result's metadata leaks information about the code path taken.

As a simple example, an attacker could use the `gas_used` information to distinguish between the following SNIP-2x execution methods with high confidence: `create_viewing_key`, `set_viewing_key`, `increase_allowance`/`decrease_allowance`, `send`/`transfer`, `revoke_permit`, and so on.

In some cases, an attacker could narrow or even deduce the range of possible values that certain arguments or storage items held during execution.


## Which contracts should implement SNIP-50?

**All contracts** should implement SNIP-50 in order to protect user privacy on Secret Network.


# Specification

## Evaporation via `gas_target`

SNIP-50 compliant contracts MUST support the option for users to include a `gas_target` field in every message. The value of the field is a `u32` that specifies the target quantity of CosmWasm gas for the contract to reach by the end of its execution.


### Request

| Name       | Type            | Description                                                                                                | optional |
|------------|-----------------|------------------------------------------------------------------------------------------------------------|----------|
| gas_target | number          | The intended amount of gas to use for the execution.                                                       |  yes     |


#### Example

The following example execution message shows the user requesting to target 40,000 GAS. Even if the contract only uses 32,000 to complete the transfer, it will _evaporate_ the remainder:

```json
{
  "transfer": {
    "recipient": "<address>",
    "amount": "100",
    "padding": "-------",
    "gas_target": 40000,
  },
}
```


### Response

_Response format is not specified._

### 


# Implementation Guide

The following section is provided for developers' reference.

## API functions

Secret Network make two API functions available to contracts that allows for evaporation.


#### `check_gas()`

This API function returns the current amount of CosmWasm gas used as a `u64` and is called in a contract with `deps.api.check_gas()?` .


#### `gas_evaporate(amount: u32)`

This API function consumes a fixed amount of CosmWasm gas and is called in a contract with `deps.api.gas_evaporate(amount)?`, where `amount` is a `u32` value indicating the amount of gas to be used.


## Reference Implementation

The following snippets demonstrate how to add SNIP-50 to an example SNIP-2x contract:

In `src/msg.rs`:
```rust

#[derive(Serialize, Deserialize, JsonSchema, Clone, Debug)]
#[serde(rename_all = "snake_case")]
pub enum ExecuteMsg {
  Deposit {
    entropy: Option<Binary>,
    padding: Option<String>,
    /**/ gas_target: Option<u32>, /***/
  },
  Redeem {
    amount: Uint128,
    denom: Option<String>,
    entropy: Option<Binary>,
    padding: Option<String>,
    /**/ gas_target: Option<u32>, /***/
  },
  /* ... */
}

pub trait Evaporatable {
  fn get_gas_target(self) -> Option<u32>;
}

impl Evaporatable for ExecuteMsg {
  fn get_gas_target(self) -> Option<u32> {
    match self {
      ExecuteMsg::Deposit { gas_target, .. }
      | ExecuteMsg::Redeem { gas_target, .. }
      | ExecuteMsg::Transfer { gas_target, .. }
      | ExecuteMsg::Send { gas_target, .. }
      /* ... */
      | ExecuteMsg::BurnFrom { gas_target, .. } => gas_target,
      _ => None,
    }
  }
}
```


In `src/contract.rs`:
```rust
#[entry_point]
pub fn execute(deps: DepsMut, env: Env, info: MessageInfo, msg: ExecuteMsg) -> StdResult<Response> {
  /* ... */

  let response = match msg.clone() {
    ExecuteMsg::Deposit { .. } => { /* */ }
    ExecuteMsg::Redeem { .. } => { /* */ }
    ExecuteMsg::Transfer { .. } => { /* */ }
    /* ... */
  }

  /* then, at the very end of the `execute` function... */

  // get the target gas value
  let gas_target = match msg.clone().get_gas_target() {
    None => 0u32,
    Some(t) => t,
  }

  // check how much gas has been consumed so far
  let gas_used: u64 = deps.api.check_gas()?;

  // some remainder is available
  if (gas_target as u64) > gas_used {
    // calculate amount of gas to evaporate
    let to_evaporate = gas_target - gas_used as u32;

    // evaporate specified amount
    deps.api.gas_evaporate(to_evaporate)?;
  }

  // return the response
  response
}
```

### Evaporation Accuracy

Note, that there will be a small amount of wasm execution that occurs after the call to the `gas_evaporate` API function, therefore the exact amount of gas used can sometimes be off by 1 gas from the target.

Wallets can adjust the `gas_target` accordingly after testing. Alternatively, if the contract developer wishes, the final `gas_used` can be fuzzied by adjusting it with a small random offset each time.


### Callback functions

The above approach works well for contract executions that do not send submessages. However, when using the receiver interface for SNIP-20 `send` messages and similar callbacks, developers might need to send additional gas target information through the `msg` parameter depending on the use case.


### Hard-coded gas target

Alternatively, if the gas used by a contract's function calls is deterministic, a contract developer can hard-code specific minimum gas targets for execute messages. This approach is less flexible, however, and developers should make sure to add functionality to adjust the target values in case gas metering changes with a chain upgrade.
