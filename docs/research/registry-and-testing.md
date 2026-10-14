# Registry publishing and OCI module testing

Research for issue #4 (parent map #1). Date of research: 2026-10-13. Local Terraform: v1.16.4 (latest release is v1.16.5).

Method: HashiCorp developer docs, the tool repositories themselves (GitHub API and source files), and two small experiments run locally with Terraform v1.16.4 and `oracle/oci` provider 9.9.0. Claims marked **(inferred)** or **(unverified)** are not stated by a primary source.

## Summary and recommended baseline

- **Registry**: nothing beyond a public GitHub repo named `terraform-oci-<name>`, semver tags and the standard module layout. Sign-in goes through GitHub OAuth, and the org's OAuth app policy may need an owner to approve the Registry app (see [Registry requirements](#1-registry-requirements)).
- **Fast tests**: `terraform test` with `mock_provider "oci"` and `command = plan` runs is cheap and worth having. Mocks produce non-OCID ids (verified), so assertions must target configuration and plan values, not IDs, or use `mock_resource` defaults / `override_resource`.
- **Real-apply tests**: not in per-PR CI. If wanted, run them manually or nightly, in a dedicated child compartment with a compartment quota, using only Always Free shapes. Because the tenancy is Pay As You Go, nothing hard-stops spend.
- **tflint**: use the bundled `terraform` ruleset. The only OCI ruleset (`joelp172/tflint-ruleset-oci`) is a one-person, six-rule project with no release since March 2025. Do not depend on it.
- **Security scanning**: Checkov is the only scanner with real `oracle/oci` rules (22 `CKV_OCI_*` checks). Trivy's single Oracle check targets the legacy `opc` provider, so it covers nothing for OCI.
- **Docs and hooks**: `terraform-docs` in inject mode, run through `pre-commit-terraform` hooks.
- **Releases**: release-please (`terraform-module` strategy, manifest mode) driven by Conventional Commits, which the repo already uses.

## 1. Registry requirements

Source: [Publish modules](https://developer.hashicorp.com/terraform/registry/modules/publish) and [Standard module structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure).

| Requirement | Detail |
| --- | --- |
| Host and visibility | The module "must be on GitHub and must be a public repo" for the public Registry. |
| Name | `terraform-<PROVIDER>-<NAME>`; `<NAME>` may contain extra hyphens. `terraform-oci-vcn`, `terraform-oci-vps` and the rest fit. `oci` is the local name of the `oracle/oci` provider. |
| Description | A one-sentence GitHub repository description becomes the module's short description. |
| Tags | A semantic version with an optional `v` prefix (`v1.0.4` or `0.9.2`). At least one matching tag must exist before first publish. Tags that do not look like versions are ignored. |
| Layout | Root module with `main.tf`, `variables.tf`, `outputs.tf` and `README.md` recommended. Every variable and output should have a description. `LICENSE` is recommended. `modules/` holds nested modules (a nested module with a README is treated as externally usable). `examples/` holds examples whose `module` blocks should use the external source address, not a relative path. Terraform files must sit in the repo root. |
| Release flow | Push a new tag; the Registry webhook picks it up, "usually within a minute". If a version is missing, use **Resync Module**, which resyncs all versions. |
| Removal | The owner can remove one or all versions. The page does not describe deleting a module outright. |

**How an org signs in and publishes.** The page says only that you connect a GitHub account. It requests access to public repositories only, repository hooks (to register a webhook) and your email address (for module alerts). The upload form lists repos filtered by the naming convention.

**Not covered by HashiCorp's page (unverified for the Registry specifically):**

- How an org-owned repo is handled, and what the Registry namespace becomes. **(inferred)**: it is the GitHub owner, giving `clonecademy/<name>/oci`.
- GitHub's [OAuth app access restrictions](https://docs.github.com/en/organizations/managing-oauth-access-to-your-organizations-data/about-oauth-app-access-restrictions) are on by default for new orgs. Members cannot grant third-party OAuth apps access; an owner must approve them, and unapproved apps cannot perform privileged create/update/delete actions on public org resources (webhook registration would be one). **(inferred)**: an org owner must approve the Terraform Registry app (or sign in as an owner) the first time, and a member who is not an owner cannot publish.
- No "verified module" badge process is documented on the publish page.
- The Registry does not run tests or checks on publish; it only parses structure.

## 2. `terraform test` as of v1.16

Sources: [Tests](https://developer.hashicorp.com/terraform/language/tests), [Mocking](https://developer.hashicorp.com/terraform/language/tests/mocking), [`terraform test` command](https://developer.hashicorp.com/terraform/cli/commands/test).

- **Versions**: test framework since v1.6.0; mocking (`mock_provider`, `override_*`) since v1.7.0.
- **Files**: `*.tftest.hcl` / `*.tftest.json`, default directory `tests/`, run from the module root. Flags include `-filter`, `-json`, `-junit-xml`, `-verbose`, `-parallelism` (default 10) and `-test-directory`.
- **Run blocks**: `command = apply` (default) creates real infrastructure; `command = plan` does not. Other run attributes: `plan_options` (`mode`, `refresh`, `replace`, `target`), `variables`, `module`, `providers`, `assert`, `expect_failures`, `state_key`, `parallel`. Runs are sequential unless every run involved sets `parallel = true` and does not share state or outputs.
- **Cleanup**: after each file, Terraform tries to destroy what it created (reverse order) and reports anything it could not destroy. HashiCorp recommends dedicated test accounts that can be purged routinely.
- **Mocks**:
  - `mock_provider` uses the real provider schema. It generates values only for computed attributes (numbers 0, booleans false, strings random 8-character alphanumerics, collections empty).
  - `mock_resource` / `mock_data` set `defaults` per type.
  - `override_resource` / `override_data` set `values` for a target; `override_module` sets `outputs`. Run-level overrides beat file-level.
  - `override_during` chooses plan or apply timing.
  - Documented limitations: generated values rarely match real formats, and for collections one set of values applies to every instance (no per-instance values).
- **`expect_failures`**: works with custom conditions (variable validation, preconditions, checks), best with `command = plan`.
- **Changelog note**: the `main` CHANGELOG mentions experimental `backend` blocks and `skip_cleanup` for run blocks plus an experimental `terraform test cleanup` command ([CHANGELOG](https://github.com/hashicorp/terraform/blob/main/CHANGELOG.md)). I did not confirm whether these are stable in v1.16, so do not rely on them.

### What mocks cannot catch for OCI (verified locally)

Experiment with `mock_provider "oci" {}` on Terraform v1.16.4 and `oracle/oci` 9.9.0:

1. A `command = plan` run asserting `oci_core_vcn.this.cidr_blocks[0] == "10.0.0.0/16"` passed.
2. An `apply` run asserting `startswith(output.vcn_id, "ocid1.vcn.")` **failed**: the mocked id was an 8-character string (`jwqwg6xh`). Anything that parses, validates or interpolates OCIDs, or depends on id format, behaves differently under mocks. Fix with `mock_resource "oci_core_vcn" { defaults = { id = "ocid1.vcn.oc1..mock" } }`, which is still a single value shared by all instances.
3. A plan with `cidr_blocks = ["not-a-cidr"]` and `access_type = "BogusAccess"` on a bucket **passed** under the mock provider. Server-side validation is not exercised. I did not test whether the real provider rejects these at plan time; I expect most OCI validation happens at the API, so a real `plan` would not catch them either **(unverified)**.

Other gaps (these follow from how mocks work, not from an OCI-specific source): service limits and quotas, availability-domain and shape capacity ("Out of host capacity" on A1 is a real Always Free failure mode, see [Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)), IAM policy syntax and evaluation, eventual consistency and ordering between resources, defaults the API fills in, and cloud-init behavior.

**Good fit for mocks**: variable validation and `expect_failures`, conditional resources and `count`/`for_each` wiring, output shapes, locals and naming, and cross-resource references.

## 3. Real-apply integration tests in CI

Options (judgement, built on the sources named):

| Option | Cost and risk |
| --- | --- |
| `terraform test` with `command = apply` against a real tenancy | The framework supports it and cleans up on exit, but failures can leave orphans ([command docs](https://developer.hashicorp.com/terraform/cli/commands/test)). Tenancy is Pay As You Go, so a mistake in a shape or size can bill. |
| Same, restricted to Always Free shapes in a child compartment with a compartment quota | Best containment. Quotas are set by `set` / `zero` / `unset` policy statements and work alongside budgets ([Compartment quotas](https://docs.oracle.com/en-us/iaas/Content/Quotas/Concepts/resourcequotas.htm)). The Oracle page does not say how quotas interact with Always Free resources, so confirm in a test. Oracle also states that, after upgrading, you pay only for usage above Always Free limits, so quotas are the guard. |
| Separate free-tier-only tenancy | Strongest isolation but adds an account and a second set of credentials. Not recommended for ten small modules. |
| Skip real apply; rely on mock tests plus manual `examples/` applies | Cheapest. Accepts the gaps listed above. |

**Constraints that apply to any real run:**

- Always Free limits: 2 VCNs, 2 AMD micro instances, A1 up to 2 OCPUs / 12 GB total, 200 GB block storage, one MySQL HeatWave system, and so on ([Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)). Ten modules testing in parallel will collide on VCN and A1 limits, so serialise runs.
- Idle Always Free compute can be reclaimed. A1 creation can fail for lack of capacity, which makes the test flaky regardless of the module.
- Credentials: GitHub OIDC to OCI is possible via an Identity Domain identity propagation trust and the provider's token exchange. Third-party references describe it ([austenstone/terraform-oci-github-oidc](https://github.com/austenstone/terraform-oci-github-oidc), [dgutierrezcolodra/oci-terraform-github-actions-wif-example](https://github.com/dgutierrezcolodra/oci-terraform-github-actions-wif-example)). I did not find an Oracle-owned how-to for this in my searches, and the exchange still needs a client credential stored in GitHub. The tenancy-repo auth decision (a separate ticket) should settle this once for all repos.
- Fork PRs get no secrets, so real-apply jobs must be limited to same-repo branches or manual dispatch.

## 4. tflint and OCI

- **Official rulesets** in the `terraform-linters` org: `aws`, `azurerm`, `google`, `opa` and `terraform` (active, pushed within the last month). There is no OCI ruleset in the org ([org repo list via GitHub API](https://github.com/terraform-linters)).
- **Third-party**: [`joelp172/tflint-ruleset-oci`](https://github.com/joelp172/tflint-ruleset-oci). Created March 2025, releases v0.1.0 to v0.1.2 (all March 2025), last commit August 2025 (a dependabot merge), 1 star, 0BSD licence. Six rules: provider hardcoded keys, compute in-transit encryption, compute monitoring, NSG SSH open, bucket public access, bucket versioning. The README pins `0.1.1` while v0.1.2 exists. Treat as unmaintained. `amitgem/tflint-ruleset-ocirm` is empty and untouched since 2021.
- **Use instead**: the bundled [`terraform` ruleset](https://github.com/terraform-linters/tflint-ruleset-terraform) with `preset = "recommended"`. It includes `terraform_required_providers`, `terraform_required_version`, `terraform_typed_variables`, `terraform_unused_declarations` and `terraform_module_pinned_source`. Plugin config syntax is in the [plugin guide](https://github.com/terraform-linters/tflint/blob/master/docs/user-guide/plugins.md). Note `terraform_documented_variables` and `terraform_documented_outputs` are not in the recommended preset, so enable them explicitly to enforce HashiCorp's description guidance.
- Writing an own ruleset is possible with the [plugin SDK](https://github.com/terraform-linters/tflint-plugin-sdk) but is disproportionate for a ten-module family.

## 5. Static security scanners

| Scanner | OCI coverage | Evidence |
| --- | --- | --- |
| **Checkov** (v3.3.29) | **Yes.** 22 `CKV_OCI_*` Terraform checks, ids 2 to 23. | [`checks/resource/oci`](https://github.com/bridgecrewio/checkov/tree/main/checkov/terraform/checks/resource/oci) read from source. Covers block volume backup and CMK encryption, instance in-transit encryption, legacy metadata endpoint, monitoring, object storage versioning, CMK, public access and events, file system CMK, security list and NSG ingress (SSH, RDP, stateless), IAM password policy, Data Catalog public access. |
| **Trivy** (v0.75.0) | **None for `oracle/oci`.** Its one Oracle check, `OCI-0001` (CRITICAL), inspects IP address reservations in `public-ippool`, written for the legacy `opc_compute_ip_address_reservation` resource (Oracle Public Cloud Classic), not `oci_core_*`. | [`no_public_ip.rego`](https://github.com/aquasecurity/trivy-checks/blob/main/checks/cloud/oracle/compute/no_public_ip.rego) and its [`.yaml`](https://github.com/aquasecurity/trivy-checks/blob/main/checks/cloud/oracle/compute/no_public_ip.yaml) examples. Trivy still adds general Terraform checks and custom Rego is possible ([Trivy docs](https://trivy.dev/docs/latest/scanner/misconfiguration/config/config/)). |
| tflint OCI plugin | Six rules, see above. | |
| KICS, Terrascan | Not evaluated. | |

**Fit with the free-tier goal (judgement):** several Checkov rules recommend paid or heavier features, namely customer-managed keys (`CKV_OCI_3`, `CKV_OCI_9`, `CKV_OCI_15`), block volume backups (`CKV_OCI_2`; the free tier allows 5 backups) and object events (`CKV_OCI_7`). Expect to skip some with inline `#checkov:skip=` annotations and a reason. Vault allows unlimited software-protected keys at no cost ([Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)), so the CMK rules can be met in some modules via the `vault` module, at the cost of coupling.

## 6. Docs and pre-commit

- **terraform-docs** v0.24.0 (2026-05-10). Config in `.terraform-docs.yml` (`formatter` required). `--output-mode inject` rewrites only the section between `<!-- BEGIN_TF_DOCS -->` and `<!-- END_TF_DOCS -->`. There is an official [`terraform-docs/gh-actions`](https://github.com/terraform-docs/terraform-docs) action and a pre-commit hook.
- **pre-commit-terraform** v1.109.2 (2026-09-30, [repo](https://github.com/antonbabenko/pre-commit-terraform)). Hooks: `terraform_fmt`, `terraform_validate`, `terraform_docs`, `terraform_tflint`, `terraform_trivy`, `terraform_checkov`, `terraform_providers_lock` and others. `terraform_tfsec` is deprecated in favour of `terraform_trivy`. I did not see a `terraform_test` hook, so run `terraform test` in CI or as a local hook.
- tflint v0.64.0 (2026-07-17).

## 7. Release automation

- **release-please** v17.11.2 (2026-08-24), with [release-please-action](https://github.com/googleapis/release-please-action) v5.0.0 (2026-04-22) from the `googleapis` org. It has a first-class `terraform-module` release type described as "a version in the README.md, and a CHANGELOG.md" ([customizing](https://github.com/googleapis/release-please/blob/main/docs/customizing.md)). Per the [strategy source](https://github.com/googleapis/release-please/blob/main/src/strategies/terraform-module.ts) it updates `CHANGELOG.md`, `README.md`/`readme.md`, `versions.tf` (and `versions.tf.tmpl`) and `metadata.yaml`, but only if those files already exist. I could not confirm from source what exactly it rewrites in `versions.tf` or `README.md`, so check on a trial run. It opens a release PR from Conventional Commits and, on merge, tags and creates the GitHub Release.
  - Needs `contents`, `issues` and `pull-requests` write permissions, and the repo setting "Allow GitHub Actions to create and approve pull requests".
  - Events created by the default `GITHUB_TOKEN` do not trigger other workflow runs, so CI will not run on the release PR unless a PAT or GitHub App token is passed in. **(inferred)**: the Registry's repository webhook is not an Actions workflow and should still fire on the tag, with **Resync Module** as the fallback; confirm on the first release.
  - Manifest mode (`release-please-config.json`, `.release-please-manifest.json`) supports `extra-files`, custom labels and PR titles. Tag format defaults are in the strategy docs, not on the page I read; confirm the tag comes out as `vX.Y.Z` on a trial run.
- **semantic-release** v25.0.9 (2026-08-05) is the alternative. It tags straight from commits on merge with no release PR. I did not find a Terraform-specific first-party plugin, and it is Node-centric. I did not review its docs in depth.
- Either way, the repo's existing Conventional Commits rule is what drives versioning.

## 8. Recommended baseline per module repo

1. **Layout**: standard module structure, `examples/<name>/` using the external source address, `LICENSE`, one-line repo description, topics `terraform-module` and `oci`.
2. **Pre-commit** (`pre-commit-terraform`): `terraform_fmt`, `terraform_validate`, `terraform_docs` (inject mode), `terraform_tflint` (bundled `terraform` ruleset, recommended preset plus the documented-variables rules).
3. **CI on PR**: the same hooks, `terraform test` (mock provider, `command = plan` mostly, `mock_resource` defaults for OCID-shaped ids), `checkov` on the module and `examples/` with a documented skip list. Skip Trivy for OCI-specific checks.
4. **Real apply**: out of per-PR CI. If wanted later, a manual `workflow_dispatch` job in a dedicated compartment with a quota, free shapes only, serialised across repos. Decide with the tenancy auth ticket.
5. **Release**: release-please manifest with the `terraform-module` type, tags `vX.Y.Z`, first tag `v0.1.0` by hand or by config. Org owner approves the Registry GitHub app, then each repo is uploaded once through the Registry UI.

## Open questions and uncertainties

- Exact Registry behavior for org-owned repos (namespace, OAuth approval) is not in HashiCorp's docs; verify with a first publish.
- Whether the real `oracle/oci` provider rejects bad CIDRs or enum values at plan time was not tested.
- Whether compartment quotas constrain Always Free shapes as expected on a PAYG tenancy needs a test.
- release-please tag format and `versions.tf` rewrite semantics need a trial run.
- Checkov's `CKV_OCI` set was read from the `main` branch source; the published policy index was too large to fetch in full, so the list is not cross-checked against it.
- Experimental `terraform test` features (`skip_cleanup`, `backend`, `test cleanup`) are not confirmed stable in v1.16.
- KICS, Terrascan and OPA-based OCI policy sets were not evaluated.
