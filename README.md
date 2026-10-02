# AEAP Receipt Verifier — GitHub Action

**An ecosystem adapter that distributes Aiyo's canonical verification primitive into GitHub Actions workflows.**

This Action is a thin composite wrapper around [`@aiyoplane/verify`](https://www.npmjs.com/package/@aiyoplane/verify) — the single, portable verifier Aiyo publishes. The Action is not a second implementation; it is one of several surfaces (npm, PyPI, Homebrew, Docker, this Action, and the forthcoming AEAP conformance test vectors) through which the same canonical verifier reaches different runtimes. If you want to know what verification actually does, read the verifier's README. This README documents the adapter's security contract — what is pinned, what fails closed, and what the Action's outputs mean.

Aiyoplane, Inc. · Apache 2.0 licensed · [aiyoplane.com](https://aiyoplane.com)

Sister adapters distributing the same primitive:
- [`@aiyoplane/verify`](https://www.npmjs.com/package/@aiyoplane/verify) — npm (the canonical implementation this Action wraps)
- [`aiyoplane-verify`](https://pypi.org/project/aiyoplane-verify/) — Python port
- [`brew install aiyoplane/tap/aiyo`](https://github.com/aiyoplane/homebrew-tap) — macOS CLI
- [`aiyoplane/verify` on Docker Hub](https://hub.docker.com/r/aiyoplane/verify) — Docker image

---

## Quick start

Add this to any workflow file in `.github/workflows/`:

```yaml
- name: Verify Aiyo execution receipt
  uses: aiyoplane/verify-receipt-action@v1
  with:
    receipt: ${{ secrets.AIYO_RECEIPT }}
```

If the receipt is cryptographically authentic AND carries verdict `ALLOW`, the workflow continues. Anything else fails closed. See [Security contract](#security-contract) below.

---

## Full example — gate production deploy on a valid receipt

```yaml
name: Deploy to production

on:
  workflow_dispatch:
    inputs:
      aiyo-receipt:
        description: 'Aiyo execution receipt authorizing this deploy'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Verify Aiyo execution receipt
        id: verify
        uses: aiyoplane/verify-receipt-action@v1
        with:
          receipt: ${{ github.event.inputs.aiyo-receipt }}

      - name: Deploy (only if receipt verified AND verdict is ALLOW)
        if: steps.verify.outputs.verified == 'true' && steps.verify.outputs.verdict == 'ALLOW'
        run: |
          echo "Deploying with Aiyo authorization intent_id=${{ steps.verify.outputs.intent-id }}"
          echo "Verifier used: ${{ steps.verify.outputs.verifier-version }}"
          ./deploy.sh
```

Note the explicit `verdict == 'ALLOW'` check in addition to `verified == 'true'`. Those are separate claims — see [Why `verified` and `verdict` are separate](#why-verified-and-verdict-are-separate-outputs).

---

## Security contract

This is a security primitive. The security contract below is what the Action commits to; anything outside this list is explicitly out of scope and MUST NOT be inferred from a successful run.

### What this Action verifies

When `verified` is `"true"`, the Action has confirmed all of the following against the receipt:

1. **Signature authenticity.** The receipt's cryptographic signature validates against the issuer's public key.
2. **Issuer identity.** The signing key is published in the JWKS at the configured `jwks-url` and resolves to a known Aiyo issuer. The default JWKS URL is `https://api.aiyoplane.com/.well-known/aiyo-jwks.json`.
3. **Receipt structure.** The receipt is well-formed per Aiyo's receipt format (v1 HMAC or v2 Ed25519), with all required claims present and syntactically valid.
4. **Temporal validity.** The receipt's `exp` claim has not elapsed at the time of verification. (The receipt's `iat` is also present and parseable.)
5. **Version compatibility.** The receipt's version header is supported by the pinned verifier.

When `verdict` is `"ALLOW"`, the Action is reporting the authorization decision carried in the receipt. For current receipt formats (v1, v2) this is derived by convention — an issued receipt means ALLOW. When AEAP Full conformance receipts land, the `verdict` claim will be explicit in the payload for `ESCALATE` and `BLOCK` lineage, and the Action will report those values here.

### What this Action does NOT verify

Equally important. A `verified: true, verdict: ALLOW` result does NOT imply any of the following, and relying on this Action for any of them is a misuse:

1. **This Action does NOT call Aiyo at verification time.** Verification is offline against a fetched JWKS. The Action does not ask Aiyo "is this receipt still good right now?" Receipts carry their own expiration; use `exp` for freshness, not an API round-trip.
2. **This Action does NOT establish settlement.** A receipt means Aiyo's Runtime Decision Point authorized the action. Whether the underlying rail-level settlement completed is a separate claim carried inside the receipt's evidence block (when present). The Action verifies the receipt is authentic; it does not re-verify the on-rail settlement that may be referenced inside it.
3. **This Action does NOT verify that the subsequent workflow step succeeded.** Verifying the receipt gates whether the next step runs. It says nothing about what happens after. Monitor the outcome separately.
4. **This Action is NOT itself an authorization source.** The authorization source is Aiyo's RDP, which issued the receipt. This Action is an adapter that reads the receipt. If you trust the Action's output, you are trusting (a) the pinned verifier version, (b) the JWKS endpoint, and (c) Aiyo's RDP that issued the receipt — not the Action publisher.
5. **This Action does NOT create authority.** It consumes an authorization decision that was made elsewhere. Workflows that gate on this Action are enforcing a decision; they are not making one.
6. **This Action does NOT check revocation beyond the receipt's own `exp`.** AEAP Full conformance will introduce explicit revocation lineage; until it ships, treat the `exp` window as the revocation window.

### Fail-closed by default

The default `fail-on-invalid: true` is the fail-closed posture — matching AEAP's Invariant 5 (fail-closed default). The Action fails the workflow on:

- Invalid signature.
- Expired receipt.
- Receipt structure violation.
- Unknown signing key (not resolvable in JWKS).
- JWKS fetch failure.
- Missing or malformed receipt.
- Authentic receipt carrying a non-`ALLOW` verdict (ESCALATE, BLOCK).

Set `fail-on-invalid: false` only when you are deliberately running in advisory mode — for onboarding, canary deploys, or workflows that branch on the `verdict` output in a subsequent step. Advisory mode is a conscious opt-out; it is not the default.

### Verifier pinning

The Action does NOT default to `@aiyoplane/verify@latest`. The default is a specific pinned verifier version (currently `1.0.1`), chosen because this is a security primitive and workflow authors should opt in to automatic upgrades of a security primitive rather than inherit them silently.

You can override the pin with the `verify-version` input (any valid npm version specifier — exact pin, caret range, tilde range, or the string `latest` if you accept the tradeoff). The `verifier-version` output reports the exact version that ran, for audit trails.

### Trust boundaries

Running this Action means your workflow trusts:

1. The Action publisher (`aiyoplane/verify-receipt-action`) — scoped to this repo on GitHub.
2. The pinned `@aiyoplane/verify` version on the npm registry.
3. The JWKS endpoint at the configured `jwks-url` — defaults to Aiyo's production endpoint.
4. Aiyo's Runtime Decision Point as the authority that issued the receipt.

The Action does not add trust assumptions beyond these; it does not phone home, does not call any Aiyo endpoint other than the JWKS fetch, and does not persist receipts or verification results anywhere outside the GitHub Actions workflow log.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `receipt` | yes | — | The Aiyo execution receipt to verify. Either the receipt string itself or a path to a file containing it (see `receipt-source`). |
| `receipt-source` | no | `string` | One of `string` (default; `receipt` is the receipt itself) or `file` (`receipt` is a path to a file containing the receipt). |
| `jwks-url` | no | `https://api.aiyoplane.com/.well-known/aiyo-jwks.json` | Override the JWKS URL. Use for self-hosted Aiyo instances. |
| `verify-version` | no | `1.0.1` | The exact `@aiyoplane/verify` version to invoke. Pin to a specific version in your workflow if you require it. Use `latest` only after reviewing the tradeoff. |
| `fail-on-invalid` | no | `true` | Fail-closed default. Set to `false` for advisory mode. |

## Outputs

| Output | Description |
|---|---|
| `verified` | `"true"` if the receipt is cryptographically authentic (see §Security contract for scope). This is NOT the authorization decision — see `verdict`. |
| `verdict` | The authorization decision: `"ALLOW"`, `"ESCALATE"`, `"BLOCK"`, or empty on verification failure. For current receipt formats, an issued receipt means ALLOW. |
| `intent-id` | The `iid` claim from the verified receipt. |
| `context-id` | The `cid` claim from the verified receipt (empty if not present). |
| `merchant-id` | The `mid` claim from the verified receipt. |
| `issued-at` | The `iat` claim as an ISO-8601 timestamp. |
| `expires-at` | The `exp` claim as an ISO-8601 timestamp. |
| `verifier-version` | The exact `@aiyoplane/verify` version that performed the verification. Audit-trail output. |
| `error` | Error message if verification failed; empty on success. |
| `error-code` | Typed error code if verification failed. One of `RECEIPT_FORMAT`, `SIGNATURE_INVALID`, `RECEIPT_EXPIRED`, `JWKS_FETCH`, `UNKNOWN_KEY`, `UNSUPPORTED_VERSION`, `USAGE_ERROR`. |

---

## Why `verified` and `verdict` are separate outputs

A receipt can be cryptographically authentic while carrying a non-ALLOW verdict. These are two different claims and callers MUST check both.

| `verified` | `verdict` | Interpretation |
|---|---|---|
| `true` | `ALLOW` | Authentic receipt, authorization granted. Proceed. |
| `true` | `ESCALATE` | Authentic receipt, Aiyo's RDP asked for human review. Do NOT proceed in automated contexts. |
| `true` | `BLOCK` | Authentic receipt, Aiyo's RDP denied the action with auditable lineage. Do NOT proceed. |
| `false` | *(empty)* | Receipt failed cryptographic verification. Do NOT proceed. Inspect `error-code` to diagnose. |

The common error pattern this prevents: a workflow that gates only on `verified == 'true'` would run a workflow step for an authentic `BLOCK` receipt — exactly the opposite of what the RDP decided. Always gate on both.

Note: Current receipt formats (v1 HMAC, v2 Ed25519) are issued on ALLOW by convention — the receipt's existence is the ALLOW. The explicit `verdict` claim lands in AEAP Full conformance receipts, where BLOCK and ESCALATE lineage are preserved for audit. This Action is forward-compatible: when AEAP Full receipts arrive, the Action already surfaces the `verdict` output, and workflows that gate on it will handle BLOCK/ESCALATE automatically.

---

## Reading the receipt from a file

If a prior step wrote the receipt to a file, use `receipt-source: file`:

```yaml
- name: Verify Aiyo execution receipt from file
  uses: aiyoplane/verify-receipt-action@v1
  with:
    receipt: ./artifacts/aiyo-receipt.txt
    receipt-source: file
```

## Advisory mode — inspect verification results without failing the workflow

```yaml
- name: Inspect Aiyo receipt (advisory)
  id: inspect
  uses: aiyoplane/verify-receipt-action@v1
  with:
    receipt: ${{ env.AIYO_RECEIPT }}
    fail-on-invalid: false

- name: Branch on verification result
  run: |
    if [ "${{ steps.inspect.outputs.verified }}" = "true" ] && [ "${{ steps.inspect.outputs.verdict }}" = "ALLOW" ]; then
      echo "Receipt authentic and verdict=ALLOW; proceeding with full release."
    elif [ "${{ steps.inspect.outputs.verified }}" = "true" ]; then
      echo "Receipt authentic but verdict=${{ steps.inspect.outputs.verdict }}; routing to review queue."
    else
      echo "Receipt invalid (${{ steps.inspect.outputs.error-code }}); routing to safe-mode deploy."
    fi
```

## Pinning strategies

| Pattern | Example | When to use |
|---|---|---|
| Exact pin (default) | `verify-version: 1.0.1` | Default. Any change to the verifier requires an explicit workflow edit. |
| Patch range | `verify-version: ~1.0.0` | You want bug-fix patches automatically but no new minor features. |
| Minor range | `verify-version: ^1.0.0` | You want all 1.x.x releases automatically. Not recommended for high-sensitivity workflows. |
| Floating | `verify-version: latest` | You accept that any `@aiyoplane/verify` release is automatically trusted. Not recommended as a default. |

The Action's own version (`uses: aiyoplane/verify-receipt-action@v1` vs `@v1.0.0`) is a separate pinning decision; see GitHub's documentation on Action version pinning.

---

## About Aiyo

Aiyo is the settlement-verified Economic Execution Authorization plane for autonomous systems. Aiyo's Runtime Decision Point issues cryptographically-signed execution receipts every time it authorizes a consequential action. This Action is one of several ecosystem adapters that distribute Aiyo's canonical verifier (`@aiyoplane/verify`) into different runtimes — in this case, GitHub Actions workflows. **Verify First. Execute Second.**

The protocol spec Aiyo authors: [AEAP (Aiyo Execution Authorization Protocol)](https://aiyoplane.com/trust) v0.9.0-DRAFT. The published JWKS this Action reads: [api.aiyoplane.com/.well-known/aiyo-jwks.json](https://api.aiyoplane.com/.well-known/aiyo-jwks.json).

[aiyoplane.com](https://aiyoplane.com) · [Trust surface](https://aiyoplane.com/trust)

---

## License

Apache License 2.0 © 2026 Aiyoplane, Inc.
