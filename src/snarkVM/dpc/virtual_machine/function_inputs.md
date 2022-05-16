# Function Inputs

`FunctionInputs` is a structure that contains all the inputs for a function. This inputs are used in the evaluate operation.

```rust
pub struct FunctionInputs<N: Network> {
    pub(crate) caller: Caller<N>,
    pub(crate) recipient: Recipient<N>,
    pub(crate) amount: AleoAmount,
    pub(crate) record_payload: Payload<N>,
}
```

- `Caller` is an alias for `Address`. It is the address of the caller of the function (the transferring part).
- `Recipient` is an alias for `Address`. It is the address of the recipient of the function.
- `AleoAmount` is the amount of Aleo to be transferred.
- `Payload` is the record payload of the function.

More on how this is used in [operation](snarkVM/dpc/virtual_machine/operation.md).
