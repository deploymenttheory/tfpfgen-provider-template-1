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

Two further optional secrets, `TFPFGEN_APP_ID` and `TFPFGEN_APP_PRIVATE_KEY`,
belong to the pipeline's own GitHub App rather than to any `auth.method`. They
are what lets generation resume by itself after the last correction is
decided — see
[The GitHub App the auto-continuation needs](#the-github-app-the-auto-continuation-needs).

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

One run does the whole chain: validate the config, import the document, audit
the live API, revise the spec, generate the SDK, generate the provider,
verify it all, and open a pull request on a `tfpfgen/run-<id>` branch. Re-run
it whenever an input changes; a run that changed nothing opens nothing.

The chain does not always reach the end in one pass. When the audit finds the
document wrong about the live API, revision has corrections to propose, and
proposals you have not yet decided stop the run — see
[The corrections decision flow](#the-corrections-decision-flow).

## The workflows

Six workflows are stamped into `.github/workflows/`, numbered so they read in
pipeline order. Each is a thin caller; the behaviour lives in the toolkit.

| Workflow | Trigger | What it does |
|---|---|---|
| `10-generate` | dispatch | The generation chain above. Also publishes the proposed corrections and opens one pull request per entity per observation kind. |
| `20-corrections` | a correction pull request closing | Records your decision on it, and resumes generation once no correction PR is left open. |
| `30-ci` | push to `main`, pull requests | Build, vet, lint, coverage gate, and the drift gates that refuse a hand edit to a derived file. |
| `40-acceptance` | dispatch (schedule, once enabled) | Live acceptance tests against a real tenant, gated by a GitHub environment. Never on push — it spends real API quota. |
| `50-docs` | weekly, dispatch | Regenerates `docs/` from the provider schema; opens a PR only when they drifted. |
| `60-release` | a `v*` tag push | Preflight, GPG-sign and publish the release the Terraform Registry ingests. |

## The corrections decision flow

This is the one part of the pipeline that waits for you.

**Where corrections come from.** After the audit, `tfpfgen spec revise`
compiles the confirmed observations into proposed corrections — each one a
set of RFC 6902 operations against the imported document, a required
justification, and an evidence pointer to the observation that proves it.
A correction is only ever compiled from what the live API actually did.

**What is decided for you.** Observation kinds listed in `audit.auto_accept`
in `tfpfgen.yaml` skip proposal: their corrections land accepted directly,
named with an `auto-NNN-` prefix, and you never see a pull request for them.
Nothing is on that list until you put it there.

**What is asked of you.** Every other proposal becomes a **pull request**
labelled `tfpfgen-correction`, one per entity per observation kind. So the
enum widenings on one resource arrive as one decision and its immutability
claims as another, and you can accept the first while refusing the second.
Within a PR the findings stand or fall together: merging accepts all of them,
closing rejects all of them.

The body is written to be read rather than parsed. For each field it states
the request the audit made, what the document led it to expect, what the API
actually answered, and what the difference costs you if it goes unrecorded —
with the request and response bodies quoted as formatted JSON. Where every
finding in the PR came from the same exchange, that exchange is shown once at
the top instead of repeated under each field. The RFC 6902 operations and
observation IDs are folded away at the foot for anyone who wants the
mechanism.

**Re-running generation while decisions are open** refreshes them in place. An
undecided PR is rewritten to the newest evidence — same branch, same PR
number, no duplicate opened beside it — so a decision you have not answered
yet never shows you a stale reading. PRs you already merged or closed are left
alone.

**How to decide.** There are two answers and no third:

- **Accept — merge the pull request.** The merge moves the correction file
  into `spec/corrections/`, where revision applies it to produce
  `spec/revised.yaml`, the single source of truth for all generation.
- **Reject — close the pull request without merging.** A rejection marker is
  committed to `spec/corrections/rejected/<observation-id>.json`, and that
  observation is never proposed again while the marker stands. **The last
  comment on the PR becomes the recorded reason**, so leave one before you
  close — the marker is what a future maintainer reads to understand the
  refusal, and with no comment it records only "closed without merging".
  Deleting the marker is the only way to make the proposal eligible again.

`tfpfgen spec revise` hard-fails while any proposal is still pending, naming
each file. There is no ignore flag, and a correction cannot be left undecided
and forgotten.

**What happens next.** Closing the last open correction PR is the trigger:
`20-corrections` sees that none remain and resumes generation, reusing the
observations the audit already recorded. No new API calls are made — your
decisions are replayed against evidence already on disk — and the run
continues through SDK generation, provider generation and verification to
open the generated-provider PR on `tfpfgen/run-<id>`.

That auto-continuation is the only part of the flow that depends on more than
the stock `GITHUB_TOKEN`; see below. Without it, nothing is lost and nothing
is stuck — the run's log tells you the run id, and you dispatch
**Actions → generate → Run workflow** yourself with `reuse_audit_run_id` set
to the number on the `tfpfgen-run-id:` line at the foot of any correction PR
body. The run continues from the same observations.

### The GitHub App the auto-continuation needs

**Why an App is needed at all.** GitHub deliberately refuses to let a workflow
start another workflow using the stock `GITHUB_TOKEN`: a `workflow_dispatch`
made with it is accepted and silently starts no run, and a commit it authors
raises no event. The rule exists to stop a workflow looping forever on its own
output, and the corrections flow runs straight into it, because that flow's
last act is exactly such a dispatch — `20-corrections` ends by running
`gh workflow run 10-generate.yml -f reuse_audit_run_id=<id>`. With
`GITHUB_TOKEN` that call does nothing at all. So closing the decision loop
needs an identity of its own, one whose actions GitHub does treat as triggers.
A **GitHub App installed on this repository** is that identity, and the only
reason it exists here.

**Why an App rather than a personal access token.** A PAT would satisfy the
same rule, and that is the whole of its case. An App is owned by the
organisation rather than by a person, so it does not leave when they do; it is
installed on named repositories with named permissions, instead of carrying
one account's whole reach; and what sits in the secret is a private key, from
which `actions/create-github-app-token` mints an installation token afresh on
each run — valid for an hour, then worthless — rather than a long-lived
credential waiting in a secret store for somebody to remember to rotate it.
The correction PRs are then attributed to the App, so the history reads as the
pipeline acting rather than as a maintainer who did not.

**The two secrets.** The App's credentials are set as repository secrets:

| Secret | What it is |
|---|---|
| `TFPFGEN_APP_ID` | App ID of the pipeline's GitHub App. Not secret in itself — it is on the App's settings page — but the workflows read it as a secret so that its absence and the key's are one condition. |
| `TFPFGEN_APP_PRIVATE_KEY` | The PEM private key of that same App, exactly as downloaded. |

Both are optional, and both workflows test them together: the App path is
taken only when `TFPFGEN_APP_ID` and `TFPFGEN_APP_PRIVATE_KEY` are both
non-empty. Setting one alone changes nothing.

**They are not the audit's credentials.** `TFPFGEN_APP_*` is the pipeline's
own identity on GitHub. `TFPFGEN_AUTH_*` — including
`TFPFGEN_AUTH_APP_ID` and `TFPFGEN_AUTH_APP_PRIVATE_KEY` in the secrets table
above — are credentials for the API this repository audits, read only by the
audit job. The two prefixes are one letter apart in the middle and have
nothing to do with each other; setting one pair does not set the other, and
putting the audited API's App into `TFPFGEN_APP_*` would hand the pipeline a
key GitHub does not recognise.

**The permissions, and what exercises each.** Grant the App these four
repository permissions and no others:

| Permission | Exercised by |
|---|---|
| **contents: write** | Pushing each `tfpfgen/correction-<observationID>` branch in `10-generate`, and committing the rejection marker to the default branch in `20-corrections`. |
| **pull requests: write** | Opening one correction PR per proposal, and listing the open ones to decide whether any decision is still outstanding. |
| **issues: write** | Creating and applying the `tfpfgen-correction` label — labels are the issues API even on a pull request, so labelling fails without it. |
| **actions: write** | Dispatching the continuation run — the `gh workflow run` above, the thing the App exists for. |

A workflow job's own `permissions:` block bounds `GITHUB_TOKEN` only; it has
no effect on the App's token, which carries whatever the installation was
granted. These four therefore have to be granted on the App itself, not
inferred from the workflows.

**Registering and installing it.** Do this once per organisation and reuse the
same App for every provider repository — it is installed per repository, so
one registration serves all of them.

1. Register the App on the organisation — **Settings → Developer settings →
   GitHub Apps → New GitHub App**, following
   [GitHub's instructions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app).
   In the `deploymenttheory` organisation this App already exists and is
   called **`tfpfgen-pipeline`**, App ID **`4561840`**; install that one
   rather than registering a second.
2. **Untick Active under Webhook.** The pipeline never receives webhooks — it
   only ever calls the REST API — so an active webhook would only ask GitHub
   to post to a URL nothing reads.
3. Set the four repository permissions in the table above.
4. **Generate a private key** at the foot of the App's settings page. GitHub
   hands you a `.pem` once and keeps no copy of it; a key can be revoked and
   another generated at any time.
5. **Install the App on this repository** — App settings → Install App → the
   organisation → *Only select repositories*.
6. Set the two repository secrets. The key is read from the file rather than
   pasted, so it never reaches a shell history:

   ```bash
   gh secret set TFPFGEN_APP_ID --body <the App ID from its settings page>
   gh secret set TFPFGEN_APP_PRIVATE_KEY < <the .pem you downloaded>
   ```

   Delete the local `.pem` afterwards; it has no further use, and generating a
   replacement is a two-click job if it is ever needed again.

Setting the pair once at organisation level instead saves repeating step 6 for
every provider repository:

```bash
gh secret set TFPFGEN_APP_ID --org <org> --visibility selected \
  --repos terraform-provider-<name>
```

That needs `admin:org` on whatever token `gh` is authenticated with, which a
repository-scoped token does not carry. Repository secrets are the fallback
and behave identically — the workflows cannot tell which they were handed.

**What happens without the App.** The pipeline is not blocked by its absence,
and nothing is lost. Both workflows fall back to `github.token`: correction
PRs are still opened, still labelled, still carry their justification and
evidence; merging still accepts; closing still writes the rejection marker to
`spec/corrections/rejected/`. The single missing step is the automatic resume,
and the workflow says so rather than failing. `10-generate` warns at the point
it opens the PRs:

> `TFPFGEN_APP_ID/TFPFGEN_APP_PRIVATE_KEY` are unset; correction PRs are
> opened with `github.token`. They open and merge normally, but a merge
> authored by `GITHUB_TOKEN` raises no event, so the continuation run must be
> dispatched by hand.

and when the last correction is decided, `20-corrections` prints the run id to
dispatch with:

> every correction is decided. Dispatch `10-generate.yml` with
> `reuse_audit_run_id=<id>` to continue — a `GITHUB_TOKEN` dispatch starts no
> run, so install the tfpfgen App to have this happen by itself.

That is the manual dispatch described above, and it resumes from the same
observations, so it costs no extra API calls. The App buys one thing: not
having to read that notice.

### What a first run looks like

Expect a batch. On a large API the first audit has the whole surface to learn
at once, and it is normal for the first revision to propose **dozens** of
corrections, each arriving as its own pull request. This is the flow working:
one API's worth of accumulated document drift, surfaced all at once, each
claim separately evidenced and separately refusable.

Later runs are quiet by comparison — the accepted corrections are committed,
the rejected ones carry markers, and only genuinely new findings are
proposed.

If reviewing a whole batch by hand is more than you want, `audit.auto_accept`
is the lever. Put on it the observation kinds you already trust for this API
and their corrections stop asking. The kinds that assert a structural
constraint (`immutable`, `mutuallyExclusive`, `validConfiguration`) are those most
worth keeping under review, because they close doors on the generated
provider's schema. The list takes observation-kind names from a closed
vocabulary; `tfpfgen config validate` rejects an unknown one and prints the
vocabulary.

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

## What a correction can carry

How corrections are decided is
[The corrections decision flow](#the-corrections-decision-flow) above. What
they contain is worth knowing before you review one.

Every correction is RFC 6902 operations against the imported document, plus a
justification and an evidence pointer to the observation behind it. Accepted
ones live in `spec/corrections/`; a rejection leaves one JSON file in
`spec/corrections/rejected/`, shaped
`{"observationID": "…", "reason": "…", "rejectedAt": "…"}`.

Some corrections do more than fix a field. From what it learns about the API's
conditional behaviour, the audit can add `x-tfpfgen-*` extensions to the
revised spec — `x-tfpfgen-valid-when`, `-valid-configuration`, `-depends-on`,
`-mutually-exclusive` and `-required-when`, alongside eventual-consistency and
`x-tfpfgen-update-style` annotations — and generation turns each of these into
a config validator on the emitted resource.

## Releasing

Push a tag `vX.Y.Z`. The tag push triggers the release workflow, which calls
the toolkit's `60-release.yml`: preflight the tree (`go mod tidy` is a fixed
point, the provider builds), import the GPG key, and let goreleaser build,
sign and publish the multi-platform release the Terraform Registry ingests.
`terraform-registry-manifest.json` declares protocol version 6.0.

Docs are their own fixed point: the docs workflow (weekly, and on dispatch)
regenerates `docs/` from the provider schema with tfplugindocs and opens a
PR only when something drifted.
