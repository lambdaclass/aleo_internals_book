# General Overview of `VirtualMachine::execute`

When the VM executes a request it first checks the validity of it. 

A request is defined as the following struct:

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

A valid request means:
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

Once the request validity is asserted, the adequate operation is computed, that can be a noop, coinbase, transfer or evaluate operation. The response is then built with the resulting operation.

If the operation requested is noop, the response is built with the request.
If the operation requested is coinbase, the given amount is generated to a recipient. The response is built with the request, the recipient and the amount.
If the operation requested is transfer, the given amount is transferred from a caller to a recipient. It is ensured that the request caller matches the instruction caller, the starting balance of the caller is computed (the balance in the request minus the fee) and it is ensured that it is positive, the caller balance is computed (starting balance minus the amount) and it is ensure that it is positive. And the response is built with two outputs, one containing the caller and the caller balance, and another with the recipient and the amount.
If the operation requested is evaluate, it returns a response based on the current state of the virtual machine. It is ensured that the request caller matches the instruction caller, the starting balance of the caller is computed (the balance in the request minus the fee) and it is ensured that it is positive, the caller balance is computed (starting balance minus the amount) and it is ensure that it is positive. An output containing the function inputs recipient, the function inputs amount, the function inputs record payload and the program id is added to the response builder but it is not built yet. There is a difference here with the transfer operation, and it is that if the caller balance is not zero the change is refund to the caller adding an output to the response builder containing the function inputs caller and the caller balance. After that, the custom events are added to the response builder. If the request is not public the response is finally built but in the case of a public request the operation is added to the response builder and then built.

A response is structured like this:

```rust
pub struct Response<N: Network> {
    /// The ID of the transition.
    transition_id: N::TransitionID,
    /// The records being produced.
    records: Vec<Record<N>>,
    /// The record encryption randomness.
    encryption_randomness: Vec<EncryptionRandomness<N>>,
    /// A value balance is the difference between the input and output record values.
    value_balance: AleoAmount,
    /// The commitments on the input record values.
    input_value_commitments: Vec<N::ValueCommitment>,
    /// The commitments on the output record values.
    output_value_commitments: Vec<N::ValueCommitment>,
    /// The randomness used to generate the input value commitments.
    input_value_commitment_randomness: Vec<N::ProgramScalarField>,
    /// The randomness used to generate the output value commitments.
    output_value_commitment_randomness: Vec<N::ProgramScalarField>,
    /// The value balance commitment.
    value_balance_commitment: N::ValueBalanceCommitment,
    /// The events emitted from the execution.
    events: Vec<Event<N>>,
```

Once the response has been created, the tests of the input and output arithmetic circuits (input and output proofs) are computed.

An input circuit is basically in charge of generating the verification of the validity of the input variables (contained in the request). And an output circuit is basically in charge of generating the verification of the validity of the output variables (response).

<!-- TODO: Explain in depth how this happens -->

With the computed proofs an execution is computed. This execution with the request and the response will compose a transition. This transition will be pushed to the actual virtual machine state transition collection.

<!-- TODO: Explain Transitions in depth -->
