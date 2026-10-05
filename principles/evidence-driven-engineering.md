# Evidence-driven engineering

*Version 0.1 · 2026-10-05*

I use **evidence-driven engineering** as a name for a fairly simple discipline: keep the claim no bigger than the evidence behind it.

A green result is useful, but it always has a scope.

A local check says something about the code and environment that produced it. A CI result says something about one source state and one pipeline. A release candidate can add stronger evidence by binding tests to an exact source and exact artifacts. Staging can tell us something about that candidate in staging. None of those, by themselves, prove what is running in production.

The release process I work with makes those boundaries explicit. A candidate is bound to a full commit SHA, Git tree and source digest, pipeline identity, release manifest and immutable backend and frontend image digests. The browser evidence is checked against the same candidate identity. Deployment evidence is tied back to it again.

If the candidate changes, affected evidence has to be produced again.

That sounds strict until a pipeline is green for a state that is no longer the state being discussed.

## Evidence can go stale

A passing result does not become false because the branch moved. It becomes evidence for an older state.

This matters around long-running delivery work. In the release flow I use, the current candidate is checked against the current default branch before work begins and checked again before later mutation boundaries. If the branch has moved, the candidate is rejected as stale.

The useful question is not only whether evidence exists.

It is whether it still applies to the exact state we are about to change.

## Absence needs evidence too

Positive claims are usually easy to phrase: a test passed, a file exists, a response had the expected value.

Negative claims are more dangerous.

Claims such as "there are no other consumers", "there is no migration impact" or "nothing else needs changing" require a recorded procedure capable of finding the thing whose absence is being claimed. For material-risk boundaries, I require another complementary discovery method as corroboration.

A search that returns nothing is an observation. It becomes useful evidence only when its scope and limitations are understood.

Unknown is not the same thing as none.

## AI does not change the standard

I use coding agents in the same engineering process.

They can explore, implement, test and review quickly. They also work under explicit rules about current repository state, overlapping work, verification and release claims.

A local checkout is not proof of current remote state. A green pipeline on a stale base is not enough for merge readiness. Local or historical evidence cannot be promoted into a production claim.

The agent can do the work.

The evidence still decides what can be claimed about the work.

## Recovery is evidence

A rollback procedure that only exists on paper is not the same thing as a rollback path that has been exercised.

In the release flow I use, production preparation includes a mutating rollback drill for the exact candidate. The drill deploys the candidate, deliberately injects a failure after migrations, restores the previous release and verifies the resulting state. The actual production deployment remains blocked until that preparation succeeds.

That does not prove that every future failure will be recoverable.

It proves something narrower and more useful: this recovery path worked for this candidate under the conditions that were tested.

That is the level of claim I want the evidence to support.
