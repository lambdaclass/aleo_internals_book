# Output

This structure represents an output for a response.

```rust
pub struct Output<N: Network> {
    /// The address of the recipient.
    address: Address<N>,
    /// The balance of the recipient.
    value: AleoAmount,
    /// The program data of the recipient.
    payload: Option<Payload<N>>,
    /// The program that was run.
    program_id: Option<N::ProgramID>,
}
```

- `addres` of type `Address` is the address of the recipient.
- `value` of type `AleoAmount` is the balance of the recipient.
- `payload` of type `Option<Payload<N>>` is the program data of the recipient.
- `program_id` of type `Option<N::ProgramID>` is the program that was run.

Outputs can be noop, which means that the recipient will not be modified.
It has getter functions implemented to get the struct's fields and a function to return de output record, given the previous serial number.
