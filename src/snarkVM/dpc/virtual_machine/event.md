# Event

It is an enum that represents the different events that can occur in a transition state.

```rust
pub enum Event<N: Network> {
    /// Emits publicly-visible arbitrary data.
    Custom(Vec<u8>),
    /// Emits the view key for an output record at the specified index in a transition.
    RecordViewKey(u8, N::RecordViewKey),
    /// Emits the operation performed in a transition.
    Operation(Operation<N>),
}
```

As you can see it has three variants:

- `Custom`: Emits publicly-visible arbitrary data.
- `RecordViewKey`: Emits the view key for an output record at the specified index in a transition.
- `Operation`: Emits the operation performed in a transition.
