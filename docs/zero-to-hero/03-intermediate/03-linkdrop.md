---
sidebar_position: 4
sidebar_label: "Linkdrop contract"
title: "Introducing the linkdrop contract we can use"
---

import createMainnetAccount from '../assets/create-mainnet-account.png';
import createTestnetAccount from '../assets/create-testnet-wallet-account.png';

# The linkdrop contract

We're going to take a small detour to talk about the linkdrop smart contract. It's best that we understand this contract and it's purpose, then discuss calling a method on this contract.

The linkdrop contract is deployed to the accounts `testnet` and `near`, which are known as the top-level accounts of the testnet and mainnet network, respectively. 

## Testnet

There’s ✌️nothing special✌️ about these accounts.

When a user signs up for a testnet account on NEAR Wallet, they'll see this:

<img src={createTestnetAccount} width="400" />

Let's discuss how this testnet account gets created. 

Notice the new account will end in `.testnet`. This is because the account `testnet` will create a subaccount (like we learned about [earlier in this tutorial](/zero-to-hero/basics/add-functions-call#create-a-subaccount)) called `vacant-name.testnet`.

There are two ways to create this subaccount:
1. Use a full-access key for the account `testnet` to sign a transaction with the `CreateAccount` Action.
2. Have a smart contract deployed to the `testnet` account that has a Promise executing the `CreateAccount` Action. (More info about writing a [`CreateAccount` Promise](/promises/create-account).)

We could also use NEAR CLI to create a new account, as we'll show next.

## Mainnet

On mainnet, the account `near` also has the linkdrop contract deployed to it.

Using NEAR CLI, a person can create a mainnet account by calling the linkdrop contract, like shown below:

<img src={createMainnetAccount} />

The above command calls the `create_account` method on the account `near`, and would create `aloha.near` **if it's available**, funding it with 15 Ⓝ.

We'll want to write a smart contract that calls that same method. However, things get interesting because it's possible `aloha.near` is already taken, so we'll need to learn how to handle that.

## A simple callback

### The trait

At the top of the linkdrop smart contract, we define a trait that's used for cross-contract calls and callbacks.

The snippet below defines a trait for a callback to the same contract, but the same pattern is used when calling external contracts. Using `ext_self` is simply a convention and not a magic keyword.

```rust
use near_sdk::ext_contract; // This import simplified for clarity

#[ext_contract(ext_self)] // We'll use "ext_self" for the callback 
pub trait ExtLinkDrop {
    /// Callback after plain account creation.
    fn on_account_created(&mut self, predecessor_account_id: AccountId, amount: U128) -> bool;
}
```

### The `create_account` method

Next, we'll show the implementation of the `create_account` method. Note the [`#[payable]` macro](/contract-interface/payable-methods), which allows this function to accept an attached deposit. (Remember in the CLI command we were attaching 15 Ⓝ.)

```rust reference
https://github.com/near/near-linkdrop/blob/f24f2608e1558db773f2408a28849d330abb3881/src/lib.rs#L121-L144
```

The most important part of the snippet above is around the middle where there's:

```
Promise::new(…)
    .then(ext_self::on_account_created(…)
```

This translates to, "we're going to attempt to perform an Action, and when we're done, please call myself at the method `on_account_created` so we can see how that went."

:::caution This doesn't work

Not infrequently, developers will attempt to do this in a smart contract:

```rust
let creation_result = Promise::new("aloha.mike.near")
  .create_account();
// Check creation_result variable (can't do it!)
```

In other programming languages promises might work like this, but we must use callbacks instead. 
:::

### The callback

Now let's look at the callback:

```rust reference
https://github.com/near/near-linkdrop/blob/f24f2608e1558db773f2408a28849d330abb3881/src/lib.rs#L146-L159
```

This calls the private helper method `is_promise_success`, which basically checks to see that there was only one promise result, because we only attempted one Promise:

```rust reference
https://github.com/near/near-linkdrop/blob/f24f2608e1558db773f2408a28849d330abb3881/src/lib.rs#L35-L45
```

Note that the callback returns a boolean. This means when we modify our crossword puzzle to call the linkdrop contract on `testnet`, we'll be able to determine if the account creation succeeded or failed.

And that's it! Now we've seen a method and a callback in action for a simple contract.

:::tip This is important
Understanding cross-contract calls and callbacks is quite important in smart contract development.

Since NEAR's transactions are asynchronous, the use of callbacks may be a new paradigm shift for smart contract developers from other ecosystems. 

Feel free to dig into the linkdrop contract and play with the ideas presented in this section.

There are two additional examples that are helpful to look at:
1. [High-level cross-contract calls](https://github.com/near/near-sdk-rs/blob/master/examples/cross-contract-calls/high-level/src/lib.rs) — this is similar what we've seen in the linkdrop contract.
2. [Low-level cross-contract calls](https://github.com/near/near-sdk-rs/blob/master/examples/cross-contract-calls/low-level/src/lib.rs) — a different approach where you don't use the traits we mentioned.
:::

---

Next we'll modify the crossword puzzle contract to check for the signer's public key, which is how we now determine if they solved the puzzle correctly.