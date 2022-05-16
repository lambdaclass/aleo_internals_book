# Response Builder

It is a structure that is responsible for building the virtual machine response for an incoming request for a state transition.

```rust
pub struct ResponseBuilder<N: Network> {
    /// The request for a state transition.
    request: OnceCell<Request<N>>,
    /// A list of expected outputs for a state transition.
    outputs: Vec<Output<N>>,
    /// A publicly-visible field encoding events from the state transition.
    events: Vec<Event<N>>,
    /// A list of errors accumulated from calling the builder.
    errors: Vec<String>,
}
```

As you can see in its struct, it is composed of:

- The request for a state transition.
- A list of expected outputs for a state transition.
- A publicly-visible field encoding events from the state transition.
- A list of errors accumulated from calling the builder.

The functions that are implemented in the builder are:

- `add_output`: Adds an output to the list of expected outputs.
- `add_outputs`: Adds a list of outputs to the list of expected outputs.
- `add_event`: Adds an event to the list of events if the number of events is less than `N::NUM_EVENTS`.
- `build`: Finalizes the builder and returns a new instance of a `Response`.
