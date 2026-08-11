# Configuring this provider repository

Everything the toolkit needs to know about this API lives in one authored
file, **`tfpfgen.yaml`** at the repository root. Nothing else configures
generation — no dispatch form to fill in, no settings held in somebody's
memory. A setting in `tfpfgen.yaml` arrives through a pull request: it can be
reviewed, explained in a commit message, compared, and reverted.

Two more authored paths exist, both data: `spec/corrections/` (corrections to
the imported document) and `audit/inputs.json` (a rarely-needed escape hatch —
see [The audit](#the-audit)). Everything else in the tree is derived,
digest-tracked in `manifest.json`, and regenerated wholesale — see `CLAUDE.md`.

## Filling tfpfgen.yaml

The schema is owned by the toolkit's `internal/config` package and documented
key by key in the toolkit's
[docs/config.md](https://github.com/deploymenttheory/terraform-plugin-framework-codegen-1/blob/main/docs/config.md).
Unknown keys fail the decode with a did-you-mean suggestion; semantic
problems are reported all at once. In summary:

| Section | What it decides |
|---|---|
| `version` | Schema version of the file; currently `1`. |
| `provider` | `name` and `registry_namespace` — the registry identity, both lowercase DNS labels. |
| `generator` | `version` — the exact toolkit release tag (vX.Y.Z) the pipeline installs. Branches are refused. |
| `spec` | `document_url` — the http(s) URL of the upstream OpenAPI document. `tfpfgen spec import` pins it by SHA-256. |
| `sdk` | `backend` (`kiota` or `openapi-generator`, exactly one), its exact `backend_version` pin, the client type name, and include/exclude path scoping. |
| `auth` | `method` — how the audit authenticates — plus `api_key_header` or `token_url` where the method needs one. |
| `audit` | Bounds on the credentialed live-API stage: `enabled`, `base_url_override`, `name_prefix`, `max_objects`, `rate_limit_rps`, `auto_accept`. |
| `services` | `exclude` — spec entities that become no provider code. |

The committed skeleton marks every required key `FILL_ME`; `tfpfgen config
validate` — the pipeline's first job — rejects each one by name until it is
replaced.

## Secrets

Secret values never appear in `tfpfgen.yaml`. Each auth method reads a fixed
set of repository secrets — the names are a contract, not configuration. Set
only the roles your `auth.method` needs:

| `auth.method` | Required secrets |
|---|---|
| `bearer_token` | `TFPFGEN_AUTH_TOKEN` |
| `api_key_header` | `TFPFGEN_AUTH_TOKEN` |
| `basic` | `TFPFGEN_AUTH_USERNAME`, `TFPFGEN_AUTH_PASSWORD` |
| `oauth2_client_credentials` | `TFPFGEN_AUTH_CLIENT_ID`, `TFPFGEN_AUTH_CLIENT_SECRET` |
| `github_app` | `TFPFGEN_AUTH_APP_ID`, `TFPFGEN_AUTH_APP_PRIVATE_KEY` |

Releases separately need `GPG_PRIVATE_KEY` and — when the key has one —
`GPG_PRIVATE_KEY_PASSPHRASE`; the registry requires signed checksums.

Validation checks presence without reading values; only the audit job ever
receives them. Acceptance tests use a different namespace entirely — the
`TF_<PROVIDER>_*` variables the generated provider itself reads, held in a
GitHub environment (see the acceptance workflow).

## The first dispatch

Run **Actions → generate → Run workflow**. On the first run the tree has no
pinned document yet, so pass the **`openapi_url`** input: the URL of the
API's OpenAPI document. The run pins it by hash, records the URL, and every
later run re-imports from `spec.document_url` in `tfpfgen.yaml` — the input
is never needed again.

One run does the whole chain and opens one pull request on a
`tfpfgen/run-<id>` branch: validate the config, import the document, audit
the live API, revise the spec, generate the SDK, generate the provider,
verify it all, propose the diff. Re-run it whenever an input changes; a run
that changed nothing opens nothing.

## The audit

The audit is the credentialed stage that exercises the live API to learn its
true behaviour — what it actually accepts and rejects, beyond what the
document claims. It is the only stage that touches a network or reads secret
values, and it records its findings as observations in
`audit/observations/`, each stamped with the spec hash it was observed
against.

**When it runs:** on a dispatch with `audit=true`, or automatically when no
committed observations exist yet. Otherwise the committed observations are
used as-is. A dispatch can also pass `reuse_audit_run_id` to carry forward
the observations of an earlier pipeline run instead of auditing again.

**It creates and deletes real objects.** Every object it creates carries the
`audit.name_prefix`; the run is bounded by `audit.max_objects` and
`audit.rate_limit_rps`; and cleanup deletes prefix-matched objects at the
start and end of every run. Run the audit only against a sandbox or other
non-production tenant. The toolkit does not police this — pointing it at a
disposable tenant, never production, is the operator's responsibility. On a
shared tenant that already holds foreign objects beyond the object budget the
audit refuses to start; `--force-api-audit` on `tfpfgen audit run` overrides
that refusal.

**The activity ledger.** Before each create request is sent, the audit writes
and fsyncs a line to the run's activity ledger —
`audit/runs/<runid>.activity.jsonl`, one entry per event (intent, created,
rejected, deleted). It is never committed: it records live objects in
someone's tenant. If a run crashes, `tfpfgen audit cleanup` replays the ledger
to delete by id whatever was left behind. `audit/runs/` is the audit's working
directory and is git-ignored.

**The audit is adaptive — you do not pre-seed values.** It learns each
entity's required fields from the API's own 4xx responses and borrows valid
references by reading objects that already exist, so there is normally nothing
to author. `audit/inputs.json` is a rarely-needed escape hatch, not a routine
file: it exists only for genuinely non-discoverable operational facts the
audit can neither synthesize nor read back. When it is present, three token
forms are understood:

- `${VAR}` — read from the named environment variable at execution time,
  for values that are secret or per-tenant.
- `$created:<entity>` — the id of an object the audit itself created, for
  fields that must reference a live parent.
- `<runid>` — the run-id placeholder execution substitutes into synthesised
  names.

Its absence degrades gracefully — the audit covers what it can.

## Corrections

When observations show the published document is wrong about the live API,
`tfpfgen spec revise` **proposes** corrections under
`spec/corrections/proposed/` — each one RFC 6902 operations plus a
justification and an evidence pointer to an observation. The verb
**hard-fails while any proposal is pending**, naming each file; no ignore
flag exists. Resolve every proposal in one of two ways:

- **Accept** — move the file into `spec/corrections/`. Revision applies it
  to produce `spec/revised.yaml`, the single source of truth for all
  generation.
- **Reject** — leave a marker in `spec/corrections/rejected/`: one JSON file
  naming the observation id and the reason, shaped
  `{"observationID": "…", "reason": "…", "rejectedAt": "…"}`. The proposal is
  never re-raised while the marker stands; deleting the marker is the only way
  back.

Correction categories listed in `tfpfgen.yaml`'s `audit.auto_accept` skip
`proposed/` and land accepted directly (named with an `auto-NNN-` prefix);
everything else waits for a human.

Some corrections do more than fix a field. From what it learns about the API's
conditional behaviour, the audit can add `x-tfpfgen-*` extensions to the
revised spec — `x-tfpfgen-valid-when`, `-valid-configuration`, `-depends-on`,
`-mutually-exclusive` and `-required-when`, alongside eventual-consistency and
`x-tfpfgen-update-style` annotations — and generation turns each of these into
a config validator on the emitted resource.

## Releasing

Push a tag `vX.Y.Z`. The tag push triggers the release workflow, which calls
the toolkit's `50-release.yml`: preflight the tree (`go mod tidy` is a fixed
point, the provider builds), import the GPG key, and let goreleaser build,
sign and publish the multi-platform release the Terraform Registry ingests.
`terraform-registry-manifest.json` declares protocol version 6.0.

Docs are their own fixed point: the docs workflow (weekly, and on dispatch)
regenerates `docs/` from the provider schema with tfplugindocs and opens a
PR only when something drifted.
