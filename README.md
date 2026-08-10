# tfpfgen provider template

Template for a [tfpfgen](https://github.com/deploymenttheory/terraform-plugin-framework-codegen-1)
generated provider repository. A repo stamped from it becomes a complete
terraform-plugin-framework provider — SDK, resources, datasources, docs,
registry releases — with **100% of its code generated**. You author three
data files (`tfpfgen.yaml`, `spec/corrections/`, `audit/inputs.json`);
everything else is derived from the API's OpenAPI document and regenerated
wholesale.

The template stamps identity only. Nothing here executes: the five workflows
are thin callers into the toolkit's reusable workflows at the moving major
tag (`@v0` until the 1.0.0 contract freeze, `@v1` after), so behavior
improvements reach every provider repo without touching it.

## Quickstart

1. **Create a repository from this template**, named
   `terraform-provider-<name>` — the registry, docs generation and release
   packaging all derive the provider name from the repository name.
2. **Fill in `tfpfgen.yaml`** — replace every `FILL_ME`;
   [docs/CONFIGURING.md](docs/CONFIGURING.md) explains each key.
3. **Set the repository secrets** your `auth.method` needs, plus the release
   signing pair:

   | `auth.method` | Required secrets |
   |---|---|
   | `bearer_token`, `api_key_header` | `TFPFGEN_AUTH_TOKEN` |
   | `basic` | `TFPFGEN_AUTH_USERNAME`, `TFPFGEN_AUTH_PASSWORD` |
   | `oauth2_client_credentials` | `TFPFGEN_AUTH_CLIENT_ID`, `TFPFGEN_AUTH_CLIENT_SECRET` |
   | `github_app` | `TFPFGEN_AUTH_APP_ID`, `TFPFGEN_AUTH_APP_PRIVATE_KEY` |
   | releases (always) | `GPG_PRIVATE_KEY`, `GPG_PRIVATE_KEY_PASSPHRASE` |

4. **Dispatch the pipeline** (Actions → generate → Run workflow), passing
   `openapi_url` on the first run — before the tree carries the pinned
   document.
5. **Review the pull request** the run opens on `tfpfgen/run-<id>`. The
   pipeline proposes; humans merge.

Everything else — the audit against the live API, corrections to the spec,
releasing to the Terraform Registry — is in
[docs/CONFIGURING.md](docs/CONFIGURING.md).
