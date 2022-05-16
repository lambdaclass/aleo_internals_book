# Program Public Variables

It is a structure that contains a `TransitionID`,

```rust
pub struct ProgramPublicVariables<N: Network> {
    pub transition_id: N::TransitionID,
}
```

It is used in a function execution, but the trait `Function` is not yet implemented.
