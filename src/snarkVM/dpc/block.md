# block

## Transactions
This is a Merkle Tree wrapper, in particular it is static, which means you can only build a Merkle Tree out of a list of `Transaction<N>` but you cannot add new Transactions into the tree.
And you can use it to make proofs of inclusion of the transaction in the set. This will be enough for us because once the block is created the transactions of the block will remain the same.

To be considered valid, the `Transactions` struct should:
- not be empty,
- have all valid transactions
- not have serial number duplicates
- not have commitment duplicates
- have one and only one coinbase transaction

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

## Block
The block struct is defined as follows:

```rust
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Block<N: Network> {
    /// Hash of this block.
    block_hash: N::BlockHash,
    /// Hash of the previous block.
    previous_block_hash: N::BlockHash,
    /// The block header containing the state of the ledger at this block.
    header: BlockHeader<N>,
    /// The block transactions.
    transactions: Transactions<N>,
}
```

Leaving the getters and small helper functions aside, we'd like to take a look at:
- `mine` initializes a new block from a given template
- `new_genesis` initializes a new genesis block with one coinbase transaction
- `from` initializes a new block from a given previous hash, header and transactions list
- `is_valid` is a helper function that checks whether the block is well formed
- `block_reward` is a helper function that returns the block reward for the given block height

### `mine`
The mine function is quite simple, it verifies the set of transactions in the template is not empty and after that it mines the header and together with the previous_block_hash and the transaction set it calls the `from` method to take care of the rest

### `new_genesis`
It begins by computing the coinbase transaction and using that , and the coinbase parameters (default ledger proof, block_height, block_timestamp, difficulty_target, cumulative_weight, a clean `LedgerTree`, an array with the coinbase transaction and the coinbase record). Using that it mines the block (using the `mine` method) and ensures the mined block is actually a genesis block.

### `from`
The `from` method computes the block_hash, builds the block struct using the hash and the provided parameters and ensures the built block is valid.

### `is_valid`
The `is_valid` method runs a few assertions:

- check the previous block hash is well_formed. This means that the previous block hash should point to the correct block
- checks the header is valid using the `BlockHeader::is_valid` method
- checks the transactions are valid using the `Transactions::is_valid` method
- checks the header transactions root is the same as the `Transactions` transaction root
- ensures the coinbase reward is less than the block reward and the net balance of the transactions is more than the block reward

### `block_reward`
This method calculates the reward a mined block returns.

- If this is the genesis block, it returns `N::ALEO_STARTING_SUPPLY_IN_CREDITS` credits.
- Otherwise it makes a halving calculation. 
    - It estimates the amount of blocks mined in a 3 year period (4,730,400 blocks aprox.). Let's call it \\( K \\).
    - Based on that it separates the blocks in 3:
        - Blocks \\( 1 \\) to \\( K \\): gets \\( 100 \\) CREDITS 
        - Blocks \\( K+1 \\) to \\( 2*K \\): gets \\( 100/2 \\) = \\( 50 \\) CREDITS
        - Blocks \\( 2*K+1 \\) and onwards: gets \\( 100/4 \\) = \\( 25 \\) CREDITS
