# Evidence Chain of Custody (Lab 4.4)

## What this layer proves

The GRC evidence pipeline (`.github/workflows/grc-gate.yml`) signs and immutably stores a tamper-evident record of every pull request. An assessor who has never met me can reconstruct the chain in minutes — follow an evidence reference to a vault object, run one script, read the result — without having to trust my word for any of it. The auditor does not trust; the auditor verifies.

Chain of custody is defined here as four properties: **authenticity, integrity, timeliness, and preservation**. Each is proven by a specific artifact the pipeline produces automatically, and each is checked by `scripts/verify-evidence.sh`.

## Property → proving artifact

| Property | What it asserts | Proving artifact | How it is verified |
|---|---|---|---|
| **Authenticity** | The bundle came from a specific GitHub Actions run on this repository — not from anyone holding AWS admin | Sigstore Fulcio short-lived certificate, minted from the GitHub OIDC token; the certificate carries the workflow's OIDC subject | `cosign verify-blob --bundle … --certificate-oidc-issuer https://token.actions.githubusercontent.com` |
| **Integrity** | The bundle's bytes have not changed since signing | `*.sha256` sidecar written at sign time | Recompute SHA-256 of the downloaded bundle and compare it to the sidecar |
| **Timeliness** | The signature provably existed at a knowable point in time | Rekor transparency-log entry (timestamped), packed into the `.sig.bundle` | Confirmed during `cosign verify-blob`; the Rekor entry must resolve |
| **Preservation** | The canonical evidence cannot be deleted or overwritten before retention expires | S3 Object Lock `RetainUntilDate` on the vault object | `aws s3api get-object-retention` returns a future date |

## Why signing *and* Object Lock — defense in depth

Object Lock alone (built in Lab 2.5) makes the bundle immutable, but immutability does not prove *who* created it or *when*. A determined insider with AWS admin could stand up a different bucket, drop a forged bundle, and point a careless auditor at it. The authenticity, timeliness, and integrity proofs deliberately do **not** live in my AWS account — the Fulcio certificate and the Rekor log are Sigstore's, public and append-only — so they cannot be forged from inside AWS. Conversely, signing alone proves *if* something changed but does nothing to stop the canonical copy from being deleted; Object Lock closes that gap. The two controls together cover all four properties; either one alone leaves a hole.

## Key design decision: preserve evidence even when the gate fails

In Lab 4.3 the policy gate exited the job the moment Conftest failed, which would skip the evidence upload on a red run — exactly the runs an auditor most wants to see. In this workflow the pass/fail decision is moved to a final `Gate decision` step that runs `if: always()`, after signing and vault upload. The gate still fails closed (a non-compliant PR is blocked by branch protection), but the signed evidence of *why* it failed is preserved first. Evidence is a byproduct of the build, not a casualty of a failed build.

The pipeline's OIDC role (`cgep-grc-gate`) is granted write access scoped to exactly the vault bucket and its objects (`s3:PutObject`, `s3:GetObject`, `s3:GetBucketLocation` on `arn:aws:s3:::cgep-lab-grc-evidence-vault-5f9a515c` and `/*`) — least privilege as the control, not as an afterthought.

## Reproducing the verification

For run `27316502018`:

    bash scripts/verify-evidence.sh 27316502018 \
      --vault cgep-lab-grc-evidence-vault-5f9a515c \
      --profile default

Expected result: integrity, authenticity + timeliness, and preservation each report `OK`, ending in a single line — `CHAIN INTACT for run 27316502018`.

## The tamper test: chain of custody is mathematical

Tampering a downloaded copy and re-checking demonstrates the point. A single appended byte makes the recomputed SHA-256 disagree with the sidecar, and `cosign verify-blob` rejects the file because the signature was computed over the original bytes (`bundle="502f6b4c…"` vs `payload="b665f269…"`). The mismatch is the cryptographic rejection. The vault object itself is unaffected — Object Lock blocks the overwrite, so the tampered bytes only ever live locally — and verification against the run ID still returns `CHAIN INTACT`. The chain is not aspirational; it is arithmetic.