# Working in this repository

This repository was stamped from
[`tfpfgen-provider-template-1`](https://github.com/deploymenttheory/tfpfgen-provider-template-1)
and is operated by the
[tfpfgen toolkit](https://github.com/deploymenttheory/terraform-plugin-framework-codegen-1).
It holds a generated terraform-plugin-framework provider. The template stamps
identity only; everything executable — the CLI, the reusable workflows, the
templates — lives in the toolkit and reaches this repo by pinned tag.

## 100% of the code here is generated

There are no hand-owned code files. Never hand-edit a `.go` file, any file
carrying a DO-NOT-EDIT header, or any path `manifest.json` lists as derived.
CI refuses a hand edit to a derived file, and the next generation run would
overwrite it anyway — a hand edit is at best wasted and at worst silently
lost.

To change what gets generated, change an input:

- **`tfpfgen.yaml`** — the config: provider identity, spec URL, SDK backend,
  auth method, audit bounds, service exclusions.
- **`spec/corrections/`** — committed corrections to the imported OpenAPI
  document, each one RFC 6902 operations plus a justification.
- **`audit/inputs.json`** — operator-supplied values the audit cannot
  synthesize.
- **The toolkit** — when the generated output itself is wrong, the fix
  belongs in `terraform-plugin-framework-codegen-1`, never as a local patch
  here.

## Corrections are resolved by humans, explicitly

`tfpfgen spec revise` hard-fails while `spec/corrections/proposed/` is
non-empty, and no flag exists to look away. Each proposed correction is
resolved in exactly one of two ways:

- **Accept** — move the file into `spec/corrections/`.
- **Reject** — leave a marker in `spec/corrections/rejected/` with a
  justification saying why the observation does not warrant the change.

## The pipeline proposes; humans merge

Generation runs open pull requests on branches named `tfpfgen/run-<id>`.
Review and merge them; never commit generated output directly, and never
merge a generation PR without reading its diff.

`generator.version` in `tfpfgen.yaml` pins the exact toolkit release the
pipeline installs. It moves only via a reviewed pull request — never as a
side effect of another change, and never to a branch name.

## Nothing executable belongs here

The workflows in `.github/workflows/` are thin callers: triggers, inputs and
a `uses:` line pointing at the toolkit's reusable workflows by major tag. Do
not add steps, scripts, or logic to them. A behavior change belongs in the
toolkit, where it ships to every provider repo at once through the moving
tag.
