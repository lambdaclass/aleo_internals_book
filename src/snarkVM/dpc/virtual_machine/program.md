# Program

A program defines all possible state transitions for a record.

```rust
pub struct Program<N: Network> {
    tree: MerkleTree<N::ProgramIDParameters>,
    functions: HashMap<N::FunctionID, (u8, Arc<dyn Function<N>>)>,
    last_function_index: u8,
}
```

When a program is created, a new functions merkle tree is initialized, and adds all functions to the tree.

A program has an unique ID, knows the functions it contains, can return a function given a function ID or a function index (if it exists) and can return its path (the merkle path for a given function ID).
