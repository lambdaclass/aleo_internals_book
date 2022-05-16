# Request

It is a structure that represents a request for a transaction to the virtual machine.

```rust
pub struct Request<N: Network> {
    /// The records being consumed.
    records: Vec<Record<N>>,
    /// The inclusion proofs of ledger-consumed records.
    ledger_proofs: Vec<LedgerProof<N>>,
    /// The operation being performed.
    operation: Operation<N>,
    /// The network fee being paid.
    fee: AleoAmount,
    /// The signatures for the request (each record will have one).
    signatures: Vec<N::AccountSignature>,
    /// The visibility of the operation.
    is_public: bool,
}
```

- `records`: The records being consumed.
- `ledger_proofs`: The inclusion proofs of ledger-consumed records (there is one for every record in `records`).
- `operation`: The operation being performed.
- `fee`: The network fee being paid.
- `signatures`: The signatures for the request (each record will have one). The signatures are messages signed by the caller's private key. The message is a sequence of little endian bytes of a record commitment and a record program id (it seems that in the future it will contain also the operation id and the fee).
- `is_public`: The visibility of the operation.

There are different ways of creating a request:

- `new_coinbase`: This kind of request only requires the recipient, the amount, and to know if the operation is public. There are no records to be consumed, no inclusion proofs and the operation is computed internally and it corresponds to a `Operation::Coinbase`. The fee is computed internally too (0 - amount).
- `new_transfer`: This kind of request requires a caller (that is a reference to a `PrivateKey`), a collection of the records to be consumed, the ledger proofs, the recipient address, the amount to transfer and to know if the operation is public. The operation is computed internally and it corresponds to a `Operation::Transfer`. The fee is computed internally too (total balance - amount, and the total balance is the cumulative sum of every record's amount).
- `new_noop`: This kind of request only requires the ledger proofs to be
- `new`: This is the base function that initializes a transaction `Request`, all the functions above call this one. It gets the caller address from the caller's private key, it ensures that there is at least a record to be consumed, and then sign all the records with the caller's private key.
- `from`: Returns an instance of a request with the given records, ledger proofs, operation, fee, signatures and operation visibility.

In addition to this creation functions we have getters for every field of the request struct and besides this getters we can get the balance of the caller (by computing the cumulative sum of every record's amount), the program id, the serial numbers (out of the records), the input commitments (also out of the records), and the ledger root (out of the ledger proofs).

Finally we have the `is_valid` function. For a transaction request to be valid it means that:

- the amount of records it contains is less than a predefined constant (`N::NUM_INPUTS`),
- the number of ledger proofs is the same as the number of records,
- all records have the same owner that same owner is the one who made the request,
- the records contain a value that is at least the value of the fee,
- the total value is equivalent to the sum of the aleo amount plus the fee,
- the records vector is not empty,
- if it does not contain a program id then it does not contain a function id,
- if it contains a program id then the function id is in the specific program, 
- the record commitments are included in the ledger proof,
- the signatures are valid.
