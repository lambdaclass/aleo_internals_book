# block

## Transactions
This is a Merkle Tree wrapper, in particular it is static, which means you can only build a Merkle Tree out of a list of `Transaction<N>` but you cannot add new Transactions into the tree.
And you can use it to make proofs of inclusion of the transaction in the set. This will be enough for us because once the block is created the transactions of the block will remain the same.

## BlockHeader
```rust 
pub struct BlockHeader<N: Network> {
    /// The Merkle root representing the blocks in the ledger up to the previous block
    previous_ledger_root: N::LedgerRoot,
    /// The Merkle root representing the transactions in the block
    transactions_root: N::TransactionsRoot,
    /// The block header metadata
    metadata: BlockHeaderMetadata,
    /// Nonce for Proof of Succinct Work
    nonce: N::PoSWNonce,
    /// Proof of Succinct Work
    proof: PoSWProof<N>,
}
```

The `LedgerRoot`, `TransactionsRoot` , `PoSWNonce` and `PoSWProof` mean the same you're thinking about. And the `BlockHeaderMetadata` is defined as:

```rust 
pub struct BlockHeaderMetadata {
    /// The height of this block - 4 bytes.
    height: u32,
    /// The block timestamp is a Unix epoch time (UTC) (according to the miner) - 8 bytes
    timestamp: i64,
    /// The difficulty target for this block - 8 bytes
    difficulty_target: u64,
    /// The cumulative weight up to this block (inclusive) - 16 bytes
    cumulative_weight: u128,
}
```

To be honest, the `BlockHeaderMetadata` is a glorified Dictionary/Map but it knows how to create the genesis block header and implements the `ToBytes` trait.

Let's see the `BlockHeader` (interesting) methods:

- `pub fn from(N::LedgerRoot, N::TransactionsRoot, BlockHeaderMetadata, N::PoSWNonce, PoSWProof<N>) -> Result<Self, BlockError>` is a constructor, and it also verifies the block header is well formed.
- `pub fn is_valid(&self) -> bool`. It makes some assertions
- `pub fn mine` and `pub fn mine_once_unchecked`: generates a new block and ensures its validity with the `is_valid` method (the unchecked version avoids doing this validation and calls the `prove_once_unchecked` method instead of the `mine` method).
- `pub fn to_header_inclusion_proof`: it generates a proof from the block header tree.

Back to the `is_valid` method, the assertions it makes are:
-
    - The previous ledger root cannot be the `Default::default` value.
    - The transactions root cannot be the `Default::default` value. 
    - The nonce cannot be the `Default::default` value. 
    - Either the height is 0 and the block is the genesis block or the timestamp is greater than 0 and the PoSW proof is valid.

This last item makes a call to `PoSW<N: Network>::verify_from_block_header(&self, &BlockHeader<N>) -> bool` which in turn calls the `verify` function in the same implementation using the header's difficulty target and proof, and the header root and nonce as inputs.

`fn verify(&self, difficulty_target: u64, inputs: &[N::InnerScalarField], proof: &PoSWProof<N>) -> bool` runs a few checks (PoSW difficulty target must be met, the proof type should not be "not hiding")

**question**: what is *not hiding*
