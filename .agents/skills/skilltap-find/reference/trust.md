# Trust Tiers

**TRUST IS INFORMATIONAL. It never blocks an install.**

The security policy — controlled by `[security].scan` and `[security].on_warn` in `config.toml` — determines whether an install proceeds. Trust tiers are metadata that describe the provenance of a skill or plugin. An `unverified` skill can install successfully; a `provenance` skill can still be blocked if the security scanner flags its content.

## Tiers

| Tier | Display | Source of trust |
|---|---|---|
| `provenance` | `✓ provenance` (green) | npm Sigstore Trusted Publishing OR GitHub Artifact Attestations — cryptographically verified build chain. |
| `publisher` | `● publisher` (dim) | Identity known (npm username, GitHub owner) but not cryptographically verified. |
| `curated` | `◆ curated` (dim) | Skill comes from a tap whose entry has `trust.verified: true` in `tap.json`. |
| `unverified` | `○ unverified` (dim) | No signal — no attestation, no known publisher, not from a verified tap. |

Tier ordering by assurance: `provenance` > `publisher` > `curated` > `unverified`.

## What trust signals, what it doesn't

- `provenance` means you can verify that the published artifact was built from a specific commit by a specific workflow. It does NOT mean the code is safe.
- `curated` means a tap maintainer has explicitly marked the skill as known and intentional. It does NOT mean the maintainer audited the code.
- `unverified` means no one has vouched for the source. It is not a warning — most community skills start here.

Trust narrows the question "where did this come from?" Security scanning (`on_warn = "fail"` etc.) narrows "is the content suspicious?"

## TrustInfo schema

Stored on each installed skill / plugin record in `state.json`. Only fields relevant to the tier are populated.

```typescript
const TrustInfoSchema = z.object({
  tier: z.enum(["provenance", "publisher", "curated", "unverified"]),
  npm:    z.object({ publisher: z.string(), verifiedAt: z.string() }).optional(),
  github: z.object({ verified: z.boolean(),  repo: z.string()       }).optional(),
  tap:    z.object({ verified: z.boolean(),  verifiedBy: z.string().optional() }).optional(),
}).optional()
```

```json
{
  "tier": "provenance",
  "npm": {
    "publisher": "alice",
    "verifiedAt": "2025-03-01T12:00:00Z"
  }
}
```

```json
{
  "tier": "curated",
  "tap": { "verified": true, "verifiedBy": "skilltap-skills-maintainers" }
}
```

Field population rules:
- `npm` — present when the source is an npm package and Sigstore attestation was verified.
- `github` — present when the source is a GitHub repo with Artifact Attestations.
- `tap` — present when installed from a tap whose entry has `trust.verified: true`.

## Querying trust on an installed item

```bash
skilltap info <name> --json
```

The JSON output includes the full `trust` object from `state.json`. Read `trust.tier` for the tier and the relevant sub-object (`trust.npm`, `trust.github`, `trust.tap`) for detail.

## Trust glob — different concept

Don't confuse trust **tier** with the `[security].trust` glob list. They serve different purposes:

| Concept | What it is | Effect |
|---|---|---|
| Trust **tier** (this page) | Per-record metadata describing provenance. | Display only. Never blocks an install. |
| `[security].trust` (config) | Glob array matched against tap name OR source URL. | Sources matching any glob skip the security scan entirely. |

Use `[security].trust = ["github.com/my-corp/*", ...]` to opt entire orgs / hosts out of scanning. Use trust tier output (in `find`, `info`, the TUI) to decide whether to install something you don't already trust.
