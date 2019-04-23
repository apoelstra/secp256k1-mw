Threshold Signature Module
===========================

Multisignatures are described in the MuSig paper [1]. Threshold signatures
are a generalization of these in which all participants distribute "shards"
of their keys to all other participants, enabling signature creation
without all signers present.

### Terminology

A _Schnorr signature_ with public key `P` on message `m` is a pair `(s, R)`,
where `s` is a scalar, `R` is a curvepoint, which satisfies the equation

    sG = R + eP

where `e` is some hash of `P`, `R`, and the message. We will not worry about
the specific curve or hash used, but see [2] for a full specification.

A _multisignature_ is such a signature produced by `n` signers, labelled
`1` through `n`, who each contribute a pair `(s_i, R_i)` such that

    sum_i R_i = R
    sum_i s_i = s.

We refer to each `R_i` as a _partial nonce_ and each `s_i` as a _partial
signature_.

A _threshold signature_ is such a signature that may be produced by any
`k` signers out of a fixed set of `n`. Production of such a signature
requires an additional step in which each participant splits her secret
key into `n` _key shards_, one for each participant including herself.

### Key Generation Procedure

Let `n` signers each have a keypair `(x_i, X_i)` for `i` ranging from `0` to
`n-1`. We define their _combined public key_ as

    Y = sum_i µ_i*X_i = sum_i Y_i

where `µ_i = H(L || i)`, where `H` is a cryptographic hash function
and `L` is a hash of all public keys in some canonical order. We refer to
the coefficient `µ_i` as the _MuSig coefficient_ of the key, and to the
key `Y_i = µ_i*X_i` as the _tweaked public key_.

Observe that the public key `Y` will be the same for any `k-of-n` signature
with the same `n` keys `X_i`. In particular, it does not depend on the
threshold value `k`.

Key Generation works as follows:

1. All signers agree on their initial public keys `X_i` through some external
   procedure this API does not cover.
2. Each signer uses `secp256k1_musig_pubkey_combine` to compute the combined
   public key `Y`. This function also outputs a list `Y_i` of tweaked public
   keys, which will be needed to validate partial signatures.
3. Each participant splits her secret key with
   `secp256k1_thresholdsig_keysplit`. This function outputs an array of
   private _keyshards_, one for each participant, as well as an array of
   corresponding _public coefficients_. Every keyshard is sent to the
   corresponding participant along with the array of public coefficients.
4. Each participant, upon receiving a private keyshard and a set of public
   coefficients, runs `secp256k1_thresholdsig_verify_shard` to check consistency
   of these. This function outputs a modified set of participant public keys `{Z_i}`
   and the signer's modified secret key `z`, which the signer should use in place of
   the keys produced in steps 2 and 3. (Specifically, on the first invocation
   `secp256k1_thresholdsig_verify_shard` outputs keys which it updates on
   subsequent invocations, until all participants' shards and coefficients have
   been taken into account.)
5. [TODO] She signs each set of coefficients and broadcasts her signatures.
6. [TODO] each participant verifies each others' signatures.

Key Generation Flowchart:
```
Me:                  Others:

ec_pubkey_create ->
                  <- ec_pubkey_create
pubkey_combine
                     pubkey_combine

if k < n {
keysplit         ->
                  <- keysplit
verify_shard
                     verify_shard
sign_coefficient ->
                  <- sign_coefficient
verify_signature
                     verify_signature
}
```

### Signing Procedure

After the set of `k` participating signers is established each signer `i`
produces a signature as follows.

1. The signer `i` produces a nonce pair `(k_i, R_i)` and a commitment (hash)
   `C_i` to `R_i`.  This is done with
   `secp256k1_thresholdsig_session_initialize` which also initializes `k`
   `secp256k1_musig_session_signer_data` structures, one for each
   participating signer including signer `i`. She sends the commitment `C_i`
   to every other signer.
2. Once `k` nonce commitments have been collected from all participating
   signers including signer `i`, the nonce `R_i` can be obtained with
   `secp256k1_musig_session_get_public_nonce`.
3. Signer `i` sends her public nonce `R_i` to every other signer.
4. Upon receipt of another signer's public nonce, she calls
   `secp256k1_musig_set_nonce` to update that signer's data structure.
   If the public nonce does not match the precommitment, this function will fail
   and a new signing session must be started.
5. Once `k` valid public nonces have been received, she can produce a partial
   signature using `secp256k1_musig_partial_sign`. She sends this partial
   signature to someone (or everyone) for aggregation.
6. Some participant receives all the partial signatures and combines them using
   `secp256k1_musig_partial_sig_combine`. The output of this function is a
   complete signature. If the signature is invalid, at least one of the partial
   signatures was invalid. The culprit may be identified using the function
   `secp256k1_thresholdsig_partial_sig_verify` on the individual partial
   signatures.

Signing Procedure Flowchart:
```
Me:                           Others:

session_initialize C    ->
                           <- session_initialize C
musig_get_public_nonce
R                         ->
                           <- R
partial_sign              ->
                           <- partial_sign
partial_sig_verify
combine_partial_sigs
```

### Underlying Algebra

#### Multisignatures

In the case of multisignatures, i.e. `n`-of-`n` threshold signatures, the algebra
is very simple. In `secp256k1_musig_partial_sign`, all participants' public nonces
are added to get a total public nonce. Each participant computes a messagehash
using this total public nonce and signs with their tweaked secret key.

The signatures are then summed, resulting in a signature whose key is the sum
of the signers' keys.

#### Threshold Signatures

In the case of threshold signatures, construction is a bit more involved. To
recall terminology, signer `i` has tweaked secret key `y_i`, tweaked public key
`Y_i`, and the total key is `Y = sum_i Y_i` with the sum taken over all signers.

When splitting her secret key `y_i`, signer `i` calls
`secp256k1_thresholdsig_keysplit`, which outputs two things. First, a set of
_private shards_ `y_i,j`, one for each signer `j`, which satisfies the following
equation

    y_i = sum_j L_j * y_i,j                              (1)

with the sum taken over any subset of `k` or more signers. The coefficients
`L_j` depend on the subset chosen, but the shards and total key do not. The
details of determining these coefficients are given in the next section.

Second, `secp256k1_thresholdsig_keysplit` outputs a set of _public coefficients_,
which are curvepoints `P_i,n` satisfying

    y_i,j*G = sum_{n=0}^{k-1} P_i,n * j^n                     (2)

This is equation `(*)` from [3]. What `secp256k1_thresholdsig_verify_shard` does
is to check this equation for the shard `y_i,j` that signer `j` has possession
of, and also uses equation (2) to compute public shards for all signers

    Z_j = sum_i y_i,j*G                                  (3)

That is, `secp256k1_thresholdsig_verify_shard` computes the public key
equivalent of all signers' shards of all signers keys, checks that the caller
`j`'s private shards are consistent with the computed public shard `Z_j`.

Assuming all signers see the same set of coefficents `P_i,n`, they will all
compute the same set of points `Z_j` for all `j`. Signer `j` will find that
the sum of her private shards is the discrete logarithm of `Z_j`, and that in
general

    Y = sum_j L_j * Z_j                                  (4)

That is, these public points `Z_j` satisfy the same summation equation as
the individual shares. This can be easily derived:

    sum_j L_j * Z_j
        = sum_j L_j * sum_i y_i,j * G
        = sum_i [sum_j L_j y_i,j] * G
        = sum_i y_i * G
        = Y

`secp256k1_thresholdsig_session_initialize` is given the set of participating
signers. This uniquely determines a set `L_j` of Lagrange coefficients.  After
key setup, each signer's secret key is replaced with `L_j * z_j`, the sum of
their shards, and public key is replaced by `L_j * Z_j`, which can be computed
by everyone.

Each signer `j` calls `secp256k1_musig_partial_sign` to sign. When computing the
total nonce, it uses the equation

    R = sum_j R_j

where the sum is over all participating signers, and each `R_j` is a signer's
public nonce.  The signature is computed as

    s_j = k_j + (L_j * z_j) * e

Finally, in `secp256k1_musig_partial_sig_combine`, resulting in a total
signature that satisfies

    s*G = sum_j s_j * G
        = (sum_j k_j * G) + e * (sum_j L_j * z_j * G)
        = (sum_j R_j) + e * (sum_j L_j * Z_j)
        = R + e * Y

#### Lagrange Coefficients

It remains to describe in a bit more detail how the coefficients `L_j`, called
called _Lagrange coefficients_, are actually computed. Essentially, they come
from the formula for Lagrange interpolation given in [4].

When the `i`th signer calls `secp256k1_thresholdsig_keysplit`, a uniformly
random polynomial `p_i` of degree `k-1` is chosen such that `p_i(0) = y_i`.
The public coefficients `P_i,n` for all `0 <= n <= k-1` are simply the
coefficients of this polynomial multiplied by G; observe that `P_i,0 =
p_i(0)*G = Y_i`.

The private shard given to signer `j` is simply `p(j)`. Knowing this, and
the fact that `y_i = p(0)`, we see that equation (1) is a direct application
of the Lagrange interpolation formula, while equation (2) is just

    p_i(x) * G = sum_n P_i,n * x^n

evaluated at `j`.

Now, let

    p(x) = sum_i p_i(x)

where the sum is taken over all signers; then

    p(0) = sum_i p_i(0) = sum_i Y_i = Y

and equation (3) becomes

    Z_j = sum_i y_i,j*G = sum_i p_i(j)*G = p(j)*G

i.e. the `Z_j`s are just evaluations of the sum polynomial `p`, whose 0th
coefficient is the total public key. Of course Lagrange interpolation applies
to this just as it did to the `p_i`s, which gives us equation (4)

    Y = sum_j L_j * Z_j

from which we concluded that the `Z_j`s were public shards of `Y`.



[1] `https://eprint.iacr.org/2018/068`

[2] `https://github.com/sipa/bip-metas/blob/master/schnorr.mediawiki`

[3] `https://link.springer.com/content/pdf/10.1007/3-540-46766-1_9.pdf`

[4] `https://en.wikipedia.org/wiki/Lagrange_polynomial`

