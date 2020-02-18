# ADR 047: Handling evidence from light client

## Changelog
* 18-02-2020: Initial draft

## Context

If the light client is under attack (either directly - lunatic / phantom
validators or indirectly - fork on the main chain), it's supposed to halt and
send an evidence of misbehavior to the correct full node. Upon receiving an
evidence, full node should punish malicious validators (if possible).

## Decision

When a light client sees two conflicting headers (`H1.Hash() != H2.Hash()`,
`H1.Height == H2.Height`), both having 1/3+ of voting power of the currently
trusted validator set, it will submit a `ConflictingHeadersEvidence` to all
full nodes it's connected to.

```go
type ConflictingHeadersEvidence struct {
  H1 *types.SignedHeader
  H2 *types.SignedHeader
}
```

When a full node receives the `ConflictingHeadersEvidence` evidence, it should
a) validate it b) figure out if malicious behaviour is obvious (immediately
slashable) or the fork accountability protocol needs to be started.

(a): check both headers are valid (`ValidateBasic`), have
the same height and signed by 1/3+ of known validator set.

- What to do if neither H1 nor H2 was committed on the main chain?
- What if light client validator set is not equal to full node's validator set
(i.e. from full node's point of view both headers are not properly signed)

(b): download validator set from full node and intersect it with light client's
validator set

Fork accountability specification defines 5 types of attacks.

We'll start with attacks on the light client:

### F4. Phantom validators

To punish this attack, we need support for a new Evidence type -
`PhantomValidatorEvidence`.

```go
type PhantomValidatorEvidence struct {
  Header types.Header
  CommitSig types.CommitSig
}
```

Verifying evidence is easy - download validator set for height `Header.Height`
and check that validator is not present.

### F5. Lunatic validator

```go
type LunaticValidatorEvidence struct {
  Header types.Header
  CommitSig types.CommitSig
}
```

To punish this attack, we need support for a new Evidence type -
`LunaticValidatorEvidence`. This type includes a vote and a header. The header
must contain fields that are invalid with respect to the previous block, and a
vote for that header by a validator that was in a validator set within the
unbonding period. While the attack is only possible if +1/3 of some validator
set colludes, the evidence should be verifiable independently for each
individual validator. This means the total evidence can be split into one piece
of evidence per attacking validator and gossipped to nodes to be verified one
piece at a time, reducing the DoS attack surface at the peer layer.

Note it is not sufficient to simply compare this header with that committed for
the corresponding height, as an honest node may vote for a header that is not
ultimately committed. Certain fields may also be variable, for instance the
`LastCommitHash` and the `Time` may depend on which votes the proposer includes.
Thus, the header must be explicitly checked for invalid data.

For the attack to succeed, VC must sign a header that changes the validator set
to consist of something they control. Without doing this, they can not
otherwise attack the light client, since the client verifies commits according
to validator sets. Thus, it should be sufficient to check only that
`ValidatorsHash` and `NextValidatorsHash` are correct with respect to the
header that was committed at the corresponding height.

That said, if the attack is conducted by +2/3 of the validator set, they don't
need to make an invalid change to the validator set, since they already control
it. Instead they would make invalid changes to the `AppHash`, or possibly other
fields. In order to punish them, then, we would have to check all header
fields.

Note some header fields require the block itself to verify, which the light
client, by definition, does not possess, so it may not be possible to check
these fields. For now, then, `LunaticValidatorEvidence` must be checked against
all header fields which are a function of the application at previous blocks.
This includes `ValidatorsHash`, `NextValidatorsHash`, `ConsensusHash`,
`AppHash`, and `LastResultsHash`. These should all match what's in the header
for the block that was actually committed at the corresponding height, and
should thus be easy to check.

## Status

Proposed.

## Consequences

### Positive

### Negative

### Neutral

## References

* [Fork accountability spec](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md)
