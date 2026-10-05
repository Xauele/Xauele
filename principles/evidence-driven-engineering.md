# Evidence-driven engineering

*Version 0.1.1 · 2026-10-05*

> This note describes controls implemented in a private engineering workflow. The public repository does not yet contain enough material to independently reproduce those implementation claims.

Software delivery produces plenty of positive signals: tests pass, CI turns green, an image is built, staging responds correctly. The harder question is what each result actually proves, and for which state of the system.

I use **evidence-driven engineering** as shorthand for one rule: a claim should not be broader than the evidence supporting it.

This is not a replacement for existing work. SLSA already addresses verifiable software provenance and artifact verification, while in-toto provides a model for verifying software supply-chain steps, materials and products. The problem I am interested in continues after provenance has been established: evidence becomes attached to a changing system, can become stale, can be applied to a claim wider than its actual scope, and increasingly has to survive work performed by coding agents.

## Evidence belongs to a specific state

A test result is not evidence about an application in the abstract. It describes a particular state under particular conditions.

One release process I work with binds a candidate to an identity with fields shaped like this:

```text
RELEASE_SHA=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
RELEASE_GIT_TREE=bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
RELEASE_SOURCE_SHA256=cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc
RELEASE_PIPELINE_ID=1234567890
APP_IMAGE_DIGEST=sha256:dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
FRONTEND_IMAGE_DIGEST=sha256:eeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee
```

The values above are synthetic. The field names and value shapes come from the implemented release identity.

Browser verification is checked against that same candidate identity, and deployment verification continues from it. The useful property is not the number of hashes. It is continuity: evidence produced at different stages can still be shown to concern the same candidate.

This is conventional software-supply-chain thinking applied beyond the build itself.

## Valid evidence can become stale

Assume CI passes for commit A and the default branch later moves to commit B. The evidence for A has not become false; it remains evidence for A.

That distinction matters in long-running delivery work. The release flow I use checks candidate currency before production preparation and again close to later mutation boundaries. A candidate that no longer matches the current default branch is rejected as stale.

There is still a residual race here.

Production jobs are serialized, and the artifact being deployed is sealed to an exact SHA, so another production job cannot silently replace that artifact. Repository updates are not locked by the production resource lock, however. The default branch can still move after the last currency check and before the remote mutation begins. For example, a hotfix may be merged into the default branch during that interval: the sealed candidate is still exactly the artifact that was approved, but it no longer contains the newest branch state.

The exact artifact remains known, but its status as the newest authorized branch state can become stale during that interval. The TOCTOU problem is therefore reduced, not eliminated.

## Invalidation is harder than rerunning CI

The rule “any change invalidates everything” is safe but expensive. Rerunning only tests that look related has the opposite failure mode: evidence may survive even though one of its assumptions changed.

The current policy says that a candidate change invalidates affected evidence and that the affected scope has to be rediscovered and reverified. It does not contain a complete machine-generated graph mapping every possible change to every piece of evidence it invalidates.

Today, scope discovery, dependency analysis and conservative candidate-level gates carry that responsibility.

That leaves a real design problem. Under-invalidation lets stale evidence survive. Excessive invalidation eventually turns verification into ceremony expensive enough that people start looking for ways around it.

A better change-to-evidence invalidation model remains open work here.

## Absence needs a different standard

Positive observations are usually easy to state precisely: a test passed, a file exists, a runtime returned a particular value.

Negative claims are harder:

> there are no other consumers  
> there is no migration impact  
> nothing else needs changing

In the engineering policy I use, such a claim requires an executed and recorded procedure capable of detecting the thing whose absence is being asserted. The record includes the inspected inputs and boundaries, the method used and the observed result, including a genuine zero-result when applicable.

For material-risk boundaries, another complementary discovery method is required. Two differently worded searches against the same representation do not count as independent discovery. The methods need different coverage or detection mechanisms such that one can expose a relevant dependency class the other may miss.

This does not establish universal absence. It supports the claim within the recorded scope and limitations of the executed methods.

A zero-result is evidence about the search before it is evidence about the system.

## Coding agents do not change the burden of proof

Coding agents make the distinction between explanation and evidence unusually visible. They can produce an implementation and a persuasive explanation of that implementation almost simultaneously. The explanation may be useful, but it is not independent verification.

The agents I use operate under explicit rules around repository currency, overlapping work and release claims. A local checkout is not accepted as proof of current remote state, green CI on a stale base is insufficient for merge readiness, and local or historical results cannot be promoted into production evidence.

There is also a concurrency problem above the level Git normally sees. Two branches may modify different files and still implement competing solutions to the same problem. The workflow therefore treats semantic overlap as an integration concern independently of textual merge conflicts.

Faster implementation changes how much work can be attempted. It does not increase the evidentiary value of the agent's confidence.

## Production is another evidence boundary

A staging result remains a staging result.

After production deployment, the release flow verifies live identity against the approved candidate. Production smoke reads runtime release information and compares the running backend and frontend SHA and immutable image digests with the candidate manifest. Health and readiness are checked separately, with affected-path verification required when the change calls for it.

Those observations prove different things. Runtime identity establishes which artifacts are running. Health and readiness establish the conditions covered by those probes. Neither establishes correctness of every application workflow.

Production is therefore another evidence-producing environment, not a status inherited from staging.

## Recovery has a narrower scope than it first appears

Before the normal production deployment can proceed, the exact candidate goes through a mutating deploy-and-rollback drill on the production target.

The drill records the existing state, deploys the candidate, verifies runtime identity, health and schema state, injects a failure after migrations, restores the previous application release and verifies the restored runtime.

The database behaviour is important: the rollback path does **not** reverse the schema migration. The resulting schema must remain compatible with both the candidate and the previous application version. Post-rollback verification checks migration state and release fingerprints against the recorded pre-drill state.

This produces an asymmetry in the evidence.

The drill proves the path in which the candidate migration is first applied, the candidate runs on that schema, a failure occurs and the previous application version is restored against the migrated schema.

The later production deployment starts with that migration already applied. It invokes the migration step again, but the first application of the schema change happened during the drill. For the same sealed candidate, the second `migrate` invocation is expected to find no outstanding candidate migrations: the first run records the applied migrations in the migration repository, and Laravel's migrator runs only migrations that are still pending.

That narrows the concern, but does not remove the asymmetry. The real deployment verifies the already-migrated state; it does not reproduce the first application of the schema change.

There is a second boundary around data compatibility. Runtime replacement uses controlled draining and maintenance behaviour, but the required pre-deployment drill does not enable its optional mini-load mode. It therefore does not establish that the previous application version can correctly interpret every kind of business data that the candidate might write before a real rollback incident.

Schema compatibility and cross-version data compatibility are related, but they are not the same claim.

Finally, this process mutates the production target. It is not presented here as having zero user impact.

If post-rollback verification cannot be completed, the runtime is not silently declared healthy; the recovery path attempts to leave it in a fail-closed maintenance state.

The resulting claim is deliberately narrow: this candidate successfully exercised this deploy, migration and recovery path under the tested conditions. It is not general proof of arbitrary incident recovery or arbitrary cross-version data compatibility.

## Limits

Three broader limitations remain explicit:

- this is not formal verification and does not prove that unknown dependencies cannot exist;
- evidence invalidation is not derived from a complete automatic dependency model;
- following the process does not make a human or agent conclusion correct without the required observations.

The goal is narrower: keep established, stale and unverified claims distinct enough that one cannot quietly substitute for another.

## References

- [SLSA Specification v1.2](https://slsa.dev/spec/v1.2/)
- [SLSA v1.2 — Provenance](https://slsa.dev/spec/v1.2/provenance)
- [SLSA v1.2 — Verifying artifacts](https://slsa.dev/spec/v1.2/verifying-artifacts)
- [in-toto — Getting Started](https://in-toto.io/docs/getting-started/)
