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

## Address
This is a struct representing the address for an account containing the corresponding key.
```rust 
pub struct Address<N: Network>(<N::AccountEncryptionScheme as EncryptionScheme>::PublicKey);
```

It has different ways to create an address. From a private key, from a compute key and from a view key.
The functions `from_private_key`, `from_compute_key` and `from_view_key`. It also has implementations for `From` trait with all the different keys.

**question:** Is this really necessary? we can keep the functions or the from trait.

The implementation of the one that receives the private key just transform that key to a compute key and then generates the encryption key that the address will use.
Otherwise the one that receives the view key generates a public key with that view key and use that as the address that the address will use.

The address can be created with a string too, the string being the address of that account. The address has to match some specific characteristics to be a valid address: 
- It has to be 63 characters long
- It has to be in [bech32m](https://river.com/learn/terms/b/bech32-modified/) format. It used bech32 format before, but now it's deprecated and uses it's modified version.
- The address prefix must be "aleo". 
With that specific String, it generates the the proper key associated to that address using elliptic curves cryptography.

Then there's a `verify_signature` function that verifies a signature on a message signed by the account key. This function returns true or false depending if the signature have signed a given message or false if not. This function has two parameters: 
- the first one is the message: an array of booleans representing the sequence of bits of the message that was signed.
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

## Compute key

The compute key is represented with the following struct:
```rust
pub struct ComputeKey<N: Network> {
    /// pk_sig := G^sk_sig.
    pk_sig: N::ProgramAffineCurve,
    /// pr_sig := G^r_sig.
    pr_sig: N::ProgramAffineCurve,
    /// sk_prf := RO(G^sk_sig || G^r_sig).
    sk_prf: N::ProgramScalarField,
}
```
Right now the compute key can not be created with the `new` function. That function is only for internal use. The public way to do it is from a private key with the function `from_private_key` which receives a created private key and computes the `pk_sig` and `pr_sig` to create the compute key.

It can be created with a Signature with function `from_signature` , it extracts the `pk_sig` and `pr_sig` from the signature and use that to create the correct compute key.

There are some getters that returns reference to all the fields in the struct and a function called `is_valid` that returns a boolean indicating if the compute key is well-formed or not.
Basically the validation recomputes the `sk_prf` with the `pk_sig` and `pr_sig` that are in the struct and check that matches with the `sk_prf` already saved in the compute key.

It has a function to get an encryption key from the compute key. The function `to_encryption_key` takes the `pk_sig` and `pr_sig` converts them into it's projective representation and then add them together with a `pk_prf` a scalar generated from the `sk_prf`. 

## Private Key

The private key is represented with the following struct:
```rust
pub struct PrivateKey<N: Network> {
    seed: N::AccountSeed,
    pub(super) sk_sig: N::ProgramScalarField,
    pub(super) r_sig: N::ProgramScalarField,
}
```

In this case it doesn't have the getters so the data of the struct is private except for the super crates.

It has a `new` function that receives an rng to create the seed. Then `sk_sig` and `r_sig` are calculated with the seed and two domain separators **AleoAccountSeedSignatureSecretKey0** and **AleoAccountSeedSignatureRandomizer0**.
This is used to the construction of the preimage and evaluated using [Poseidon Hash Function](https://eprint.iacr.org/2019/458.pdf)

There are a few functions that transforms the private key to another entity like `to_address`, `to_compute_key` and `to_decryption_key` that use the `sk_sig`, `r_sig` and the `sk_prf` calculated transforming the private key to a compute key.

In the function `is_valid` it transforms the private key to a compute key and calls the function mentioned above.

There is a function called `sign` that signs a message using the private key. It receives two things:
- The message: an array of booleans representing the sequence of bits of the message that is going to be signed.
- A rng to sample a random scalar field element.

The function to sign looks like this:

```rust
///
    /// Returns signature (c, s, G^sk_sig, G^r_sig), where:
    ///     c := Hash(G^sk_sig G^r_sig G^sk_prf, G^r, message)
    ///     s := r - c * sk_sig
    ///
    fn sign<R: Rng + CryptoRng>(
        &self,
        private_key: &Self::PrivateKey,
        message: &[bool],
        rng: &mut R,
    ) -> Result<Self::Signature> {
        // Sample a random scalar field element.
        let r = TE::ScalarField::rand(rng);

        // Compute G^r.
        let g_r = self.g_scalar_multiply(&r);

        // Extract (sk_sig, r_sig).
        let (sk_sig, r_sig) = private_key;

        // Compute G^sk_sig.
        let g_sk_sig = self.g_scalar_multiply(sk_sig);

        // Compute G^r_sig.
        let g_r_sig = self.g_scalar_multiply(r_sig);

        let mut to_invert = [g_sk_sig, g_r_sig, g_r];
        TEProjective::<TE>::batch_normalization(&mut to_invert);
        let [g_sk_sig_affine, g_r_sig_affine, g_r_affine] = to_invert.map(|a| a.to_affine());

        // Compute sk_prf := RO(G^sk_sig || G^r_sig).
        let sk_prf = self.hash_to_scalar_field(&[g_sk_sig_affine.to_x_coordinate(), g_r_sig_affine.to_x_coordinate()]);

        // Compute G^sk_prf.
        let g_sk_prf = self.g_scalar_multiply(&sk_prf);

        // Compute G^sk_sig G^r_sig G^sk_prf.
        let public_key = (g_sk_sig + g_r_sig + g_sk_prf).to_affine();

        // Compute the verifier challenge.
        let verifier_challenge = {
            // Construct the hash input (G^sk_sig G^r_sig G^sk_prf, G^r, message).
            let mut preimage = vec![];
            preimage.extend_from_slice(&public_key.to_x_coordinate().to_field_elements()?);
            preimage.extend_from_slice(&g_r_affine.to_x_coordinate().to_field_elements()?);
            preimage.push(TE::BaseField::from(message.len() as u128));
            preimage.extend_from_slice(&message.to_field_elements()?);

            // Hash to derive the verifier challenge.
            self.hash_to_scalar_field(&preimage)
        };

        // Compute the prover response.
        let prover_response = r - (verifier_challenge * sk_sig);

        Ok(AleoSignature {
            prover_response,
            verifier_challenge,
            root_public_key: g_sk_sig_affine.to_x_coordinate(),
            root_randomizer: g_r_sig_affine.to_x_coordinate(),
        })
    }
```

Lastly there is a `from_str` function to create a private key from a string.
The private key has to match some specific characteristics to be a valid private key: 
- It has to be 43 characters long
- It has to be in [base58](https://tools.ietf.org/id/draft-msporny-base58-01.html) scheme.
- The private key prefix must be "APrivateKey". 

