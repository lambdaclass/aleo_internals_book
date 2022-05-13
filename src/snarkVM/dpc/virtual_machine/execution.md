# Execution

```rust
pub struct Execution<N: Network> {
    pub program_execution: Option<ProgramExecution<N>>,
    pub input_proofs: Vec<N::InputProof>,
    pub output_proofs: Vec<N::OutputProof>,
}
```

It is created from a `ProgramExecution`, a vector of `InputProof` and a vector of `OutputProof`.

This structure only has two functions implemented. The one that creates it (`from`) and `verify`.

Basically what `Execution` does is verify the zero knowledge proofs attesting to the validity of a `Transition`.

The verification process is carried out as follows:

- Ensure that every input proof is valid for the given public variables with a given input verifying key.
- Ensure that every output proof is valid for the given public variables with a given output verifying key.
- Ensure that the program execution proof is valid.

This `verify` function is called by `Transition::verify`.
