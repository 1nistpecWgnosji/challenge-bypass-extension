# Deterministic methods for hashing bytes to elliptic curve points

A central portion of our design relies on deterministically hashing bytes to a
uniformly random point on an elliptic curve (we currently use the NIST P256
curve). There are two methods that we use that are controlled via the
"`hash-to-curve`" config option.

It is important that the hash-to-curve algorithm is identical in both the server
and client implementations.

## Hash-and-increment

The method that is currently used in Privacy Pass v1.0 is the
`hash-and-increment` method. Essentially this method works by hashing a
sequence of bytes (the bytes are slightly modified to match golangs point
generation procedure) and checking if the result can be decompressed into a
point on P-256. If this is successful then we can continue, otherwise we try
again using the result of the previous iteration up to 10 times before failing.

It is obvious that this provides a probabilistic (and non-negligible) chance of
failure and so this method of hashing is suboptimal. See
[here](https://github.com/privacypass/challenge-bypass-extension/blob/alxdavids/h2c/addon/scripts/crypto.js#L156-L195)
for more details.

## Affine SWU method

In Privacy Pass v1.1 we will move to supporting both the previous
`hash-and-increment` method, and a new method that we call "affine SWU". The
affine SWU method always returns a correct curve point and thus has zero chance 
of failure. This method was first described by Shallue, Woestijne and Ulas
(hence SWU). It is known as "affine" since it creates curve points in the affine
representation rather than the Jacobian representation.

The algorithm is described below. The input is a field element t that is
obtained by hashing the bytes into 𝔽_p using SHA256. The output is a coordinate
pair (x,y) for a curve point on the elliptic curve E(𝔽_p). Note that this is an
optimised version of the simplified SWU method described
[here](https://datatracker.ietf.org/doc/draft-irtf-cfrg-hash-to-curve/). We use
this optimised version as it reduces the number of exponentiations and
inversions (modulo p) that are required. See
[here](https://github.com/privacypass/challenge-bypass-extension/blob/alxdavids/h2c/addon/scripts/h2c.js#L96-L125)
for our implementation.

```
INPUT: t ∈ 𝔽_p such that t ∉ {−1,0,1}
OUTPUT: (x,y) ∈ E(𝔽_p)
1. u = −t^2
2. t0 = u^2+u
3. t0 = 1/t0
4. x = (1+t0)(−B/A)
5. g = x^2
6. g = g+A
7. g = gx
8. g = g+B
9. y = g^(p+1/4)
10. t0 = y^2
11. IF t0 ≠ g THEN 
12. x = ux
13. y = (−1)^(p+1/4)uty
14. ENDIF 
15. RETURN (x,y)
```