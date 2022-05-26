# Operation

The operation enum has four variants.

```rust
pub enum Operation<N: Network> {
    /// Noop.
    Noop,
    /// Generates the given amount to the recipient address.
    Coinbase(Recipient<N>, AleoAmount),
    /// Transfers the given amount from the caller to the recipient address.
    Transfer(Caller<N>, Recipient<N>, AleoAmount),
    /// Invokes the given records on the function and inputs.
    Evaluate(N::FunctionID, FunctionInputs<N>),
}
```

This enum represents the different operations that can be performed on the virtual machine.

A Noop operation is a placeholder for when the virtual machine is not executing any operations (noop = no operation).

A coinbase operation generates the given amount to the recipient address. It contains the recipient address and the amount to be generated. On it, the given amount is generated to a recipient.

A transfer operation transfers the given amount from the caller to the recipient address. It contains the caller address, the recipient address and the amount to be transferred. On it the given amount is transferred from a caller to a recipient.

An evaluate operation invokes the given records on the function and inputs. It contains the function ID, the function inputs and the records to be invoked.

An operation can be turned into field elements.
