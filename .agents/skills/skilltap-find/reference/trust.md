# Trust Tiers

**TRUST IS INFORMATIONAL. It never blocks an install.**

The security policy — controlled by `security.<mode>.scan` and `security.<mode>.on_warn` in config — determines whether an install proceeds. Trust tiers are metadata that describe the provenance of a skill. An `unverified` skill can install successfully; a `provenance` skill can still be blocked if the security scanner flags its content.

## Tiers

| Tier | Display | Source of trust |
|------|---------|-----------------|
| `provenance` | `✓ provenance` (green) | npm Sigstore Trusted Publishing **or** GitHub Artifact Attestations — cryptographically verified build chain |
| `publisher` | `● publisher` (dim) | Identity known (npm username, GitHub owner) but not cryptographically verified |
| `curated` | `◆ curated` (dim) | Skill comes from a tap whose entry has `trust.verified: true` in `tap.json` |
| `unverified` | `○ unverified` (dim) | No signal — no attestation, no known publisher, not from a verified tap |

Tier ordering by assurance: `provenance` > `publisher` > `curated` > `unverified`.

## What trust signals, what it doesn't

- `provenance` means you can verify that the published artifact was built from a specific commit by a specific workflow. It does NOT mean the code is safe.
- `curated` means a tap maintainer has explicitly marked the skill as known and intentional. It does NOT mean the tap maintainer audited the code.
- `unverified` means no one has vouched for the source. It is not a warning — most community skills start here.

Trust narrows the question "where did this come from?" Security scanning (`on_warn: fail` etc.) narrows "is the content suspicious?"

## TrustInfo schema

Stored on each installed skill record in `installed.json`. Only fields relevant to the tier are populated.

```json
{
  "tier": "provenance",
  "npm": {
    "publisher": "alice",
    "sourceRepo": "https://github.com/alice/my-skill",
    "buildWorkflow": ".github/workflows/publish.yml",
    "transparency": "https://search.sigstore.dev/?logIndex=...",
    "verifiedAt": "2025-03-01T12:00:00Z"
  },
  "github": {
    "owner": "alice",
    "repo": "my-skill",
    "workflow": ".github/workflows/publish.yml",
    "verifiedAt": "2025-03-01T12:00:00Z"
  },
  "publisher": {
    "name": "alice",
    "platform": "npm"
  },
  "tap": "home"
}
```

Field population rules:
- `npm` — present when `tier = "provenance"` and the source is an npm package with Sigstore attestation.
- `github` — present when `tier = "provenance"` and the source is a GitHub repo with Artifact Attestations.
- `publisher` — present when `tier >= "publisher"` (i.e. provenance or publisher).
- `tap` — present when installed from a tap; holds the tap name.

## Querying trust on an installed skill

```bash
skilltap skills info <name> --json
```

The JSON output includes the full `trust` object from `installed.json`. Read `trust.tier` to get the tier and the relevant sub-object (`trust.npm`, `trust.github`, etc.) for detail.
