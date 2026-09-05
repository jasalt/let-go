# Cryptographic primitives proposal

This branch is intentionally isolated for adding generic cryptographic runtime
primitives without application authentication policy.

Proposed public forms:

```clojure
(crypto/random-bytes n)
(crypto/constant-time-equal? left right)
```

`random-bytes` should use Go's `crypto/rand`; `constant-time-equal?` should use
`crypto/subtle.ConstantTimeCompare`, returning false for unequal lengths. Both
forms are useful for protocols such as PKCE and local bearer-key verification,
but neither should know about OAuth, Codex, or HTTP routing.

The existing `random-uuid` uses cryptographically secure UUID generation and
`hash/sha256` is already available. This proposal fills only the byte-oriented
randomness and comparison gaps.
