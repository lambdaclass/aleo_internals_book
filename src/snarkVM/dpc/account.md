# Account

## Inside Account
- account_format
- account
- address
- compute_key
- private_key
- view_key

## Account
The account is an struct containing the following fields
```rust 
pub struct Account<N: Network> {
    private_key: PrivateKey<N>,
    view_key: ViewKey<N>,
    address: Address<N>,
}
```

This can be created with the `new` function that receives an rng (random number generator) and create a new account from a new `PrivateKey`.

It has some getters for every field of the account struct `private_key`,`view_key` and `address`. The first ones returns a reference and the address one returns an owned address.

The account implements the `From` trait too, that receives an already created `PrivateKey` to create an account. This would create the address and view key from that private key and then return a new account with that data.

In the end there are `Debug` and `Display` implementations for the account, with the following format 
`Account {{private_key: "", view_key: "", address: ""}}`

## Address
This is a struct representing the address for an account containing the corresponding key.
```rust 
pub struct Address<N: Network>(<N::AccountEncryptionScheme as EncryptionScheme>::PublicKey);
```

It has different ways to create an address. From a private key, from a compute key and from a view key.
The functions `from_private_key`, `from_compute_key` and `from_view_key`. It also has implementations for `From` trait with all the different keys.

**question:** Is this really necessary? we can keep the functions or the from trait.

The implementation of the one that receives the private key just transform that key to a compute key and then generates the encryption key that the address will use.
Otherwise the one that receives the view key generates a public key with that view key and use that as the address that the addres will use.

It has a `verify_signature` function that verifies a signature on a message signed by the account key. This function returns true or false depending if the signature have signed a given message or false if not. This function has two parameters: 
- the first one is the message: an array of bools representing the sequence of bits of the message that was signed.
- The other it's the signature that we want to verify.
This function calls a verify function defined in signature module that looks like this:

```rust 
///
    /// Verifies (c == c') && (public_key == G^sk_sig G^r_sig G^sk_prf) where:
    ///     c' := Hash(G^sk_sig G^r_sig G^sk_prf, G^s G^sk_sig^c, message)
    ///
    fn verify(&self, public_key: &Self::PublicKey, message: &[bool], signature: &Self::Signature) -> Result<bool> {
        // Extract the signature contents.
        let AleoSignature { prover_response, verifier_challenge, root_public_key, root_randomizer } = signature;

        // Recover G^sk_sig.
        let g_sk_sig = Self::recover_from_x_coordinate(root_public_key)?;

        // Compute G^sk_sig^c.
        let g_sk_sig_c = self.scalar_multiply(g_sk_sig, verifier_challenge);

        // Compute G^r := G^s G^sk_sig^c.
        let g_r = (self.g_scalar_multiply(prover_response) + g_sk_sig_c).to_affine();

        // Compute the candidate verifier challenge.
        let candidate_verifier_challenge = {
            // Construct the hash input (G^sk_sig G^r_sig G^sk_prf, G^r, message).
            let mut preimage = vec![];
            preimage.extend_from_slice(&public_key.to_x_coordinate().to_field_elements()?);
            preimage.extend_from_slice(&g_r.to_x_coordinate().to_field_elements()?);
            preimage.push(TE::BaseField::from(message.len() as u128));
            preimage.extend_from_slice(&message.to_field_elements()?);

            // Hash to derive the verifier challenge.
            self.hash_to_scalar_field(&preimage)
        };

        // Recover G^r_sig.
        let g_r_sig = Self::recover_from_x_coordinate(root_randomizer)?;

        // Compute the candidate public key as (G^sk_sig G^r_sig G^sk_prf).
        let candidate_public_key = {
            // Compute sk_prf := RO(G^sk_sig || G^r_sig).
            let sk_prf = self.hash_to_scalar_field(&[g_sk_sig.to_x_coordinate(), g_r_sig.to_x_coordinate()]);

            // Compute G^sk_prf.
            let g_sk_prf = self.g_scalar_multiply(&sk_prf);

            // Compute G^sk_sig G^r_sig G^sk_prf.
            g_sk_sig.to_projective() + g_sk_prf + g_r_sig.to_projective()
        };

        Ok(*verifier_challenge == candidate_verifier_challenge && *public_key == candidate_public_key)
    }
}
```
