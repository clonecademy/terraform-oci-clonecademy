# Terraform's native `oci` backend

Research for issue #3 (parent map #1). Feeds #17 (tenancy repo bootstrap).
Researched 2026-10-13/14 against Terraform v1.16.x (local: v1.16.4).

**Evidence labels.** Each non-obvious claim says where it comes from:
**[doc]** a vendor documentation page, **[src]** the backend's Go source in `hashicorp/terraform` (`main`, `internal/backend/remote-state/oci/`), **[run]** something I ran against local Terraform v1.16.4 with throwaway fake credentials (no real tenancy was touched), **[issue]** a GitHub issue or PR. Anything I could not verify is listed under [Uncertainties](#uncertainties-and-things-to-verify).

## Short answer

- The native `oci` backend stores state in an OCI Object Storage bucket. It was introduced in **Terraform v1.12.0 (2025-05-14)**. Oracle's own docs now call the old S3-compatible approach **deprecated** and recommend the native backend.
- For GitHub Actions, the practical authentication is an **API key** for a dedicated Terraform IAM user, supplied through environment variables. `TF_VAR_tenancy_ocid`, `TF_VAR_user_ocid`, `TF_VAR_fingerprint`, `TF_VAR_private_key_path` and `TF_VAR_region` work for both the backend and the `oracle/oci` provider [run].
- `bucket` and `namespace` are **required in the `backend` block** or passed with `-backend-config`. Environment variables cannot supply them (documented limitation, reproduced [run]).
- Locking is on by default and uses a `<key>.lock` object created with `If-None-Match: *`. Workspaces are supported, stored under `tf-state-env/<workspace>/<key>`.
- Create the bucket with **versioning enabled**, **no public access** and (optionally) a customer-managed Vault key. Do **not** use retention rules: Oracle documents that they cannot coexist with versioning. Use a lifecycle policy to cap version growth.
- Bootstrap is the standard two-step: apply with local state to create the bucket, add the `backend "oci"` block, then `terraform init -migrate-state`.

## 1. Version introduced

- Terraform **v1.12.0**, released 2025-05-14: "Added Terraform backend implementation for OCI Object Storage" (NEW FEATURES). Source: [Terraform v1.12 CHANGELOG](https://github.com/hashicorp/terraform/blob/v1.12/CHANGELOG.md) **[doc]**. Later changelogs (v1.13 to v1.17) do not mention the backend, so 1.12.0 is the introduction and I found no later breaking change to it.
- Implementation PR [hashicorp/terraform#36872](https://github.com/hashicorp/terraform/pull/36872) merged 2025-04-22 and was backported to v1.12 (#36912) **[issue]**. Tracking issue: [#34465](https://github.com/hashicorp/terraform/issues/34465).
- Oracle states the same floor: "Upgrade to Terraform version v1.12.0 or greater to use the OCI native backend" ([Using Object Storage for State Files](https://docs.oracle.com/en-us/iaas/Content/dev/terraform/object-storage-state.htm)) **[doc]**.
- Maintenance: a HashiCorp maintainer says the OCI backend "is maintained by Oracle engineers" ([#37277 comment](https://github.com/hashicorp/terraform/issues/37277)) **[issue]**. It is documented in the main Terraform docs: [Backend Type: oci](https://developer.hashicorp.com/terraform/language/backend/oci).
- Local Terraform v1.16.4 includes it; the bundled OCI Go SDK was `65.89.1` [run].

## 2. Configuration arguments

Source of truth: [Backend Type: oci](https://developer.hashicorp.com/terraform/language/backend/oci) **[doc]**, cross-checked against `ConfigSchema` in `backend.go` and `constants.go` **[src]**. The doc page omits `private_key` and `private_key_password`, which the schema has.

| Argument | Required | Notes |
|---|---|---|
| `bucket` | yes | Bucket name. Must match `^[a-zA-Z0-9_-]+$` **[src]**, so a name containing a period (legal in OCI) is rejected by the backend. |
| `namespace` | yes | Object Storage namespace of the tenancy. |
| `key` | no | State object name. Default `terraform.tfstate`. Must not start or end with `/` or contain `//` **[src]**. |
| `region` | "most configurations" | Required with `InstancePrincipal`, `InstancePrincipalWithCerts`, `ResourcePrincipal` and `SecurityToken` **[src]**. With API key it can instead come from the OCI config file or env. |
| `auth` | no | `ApiKey` (default), `InstancePrincipal`, `InstancePrincipalWithCerts`, `SecurityToken`, `ResourcePrincipal`, `OKEWorkloadIdentity`. Compared case-insensitively **[src]**. See the gotcha for `OKEWorkloadIdentity`. |
| `tenancy_ocid`, `user_ocid`, `fingerprint` | for API key | |
| `private_key_path` / `private_key` | for API key | Mutually exclusive **[src]**. `private_key` is the inline PEM (literal `\n` is converted to newlines). |
| `private_key_password` | no | Passphrase for an encrypted key. |
| `config_file_profile` | no | Profile in `~/.oci/config`. Required with `SecurityToken` **[src]**. |
| `workspace_key_prefix` | no | Default `tf-state-env`. |
| `kms_key_id` | no | OCID of a Vault master key, sent as `opc-sse-kms-key-id` on state uploads **[src]**. |
| `sse_customer_key`, `sse_customer_key_sha256`, `sse_customer_algorithm` | no | SSE-C. Key and hash must be set together; `kms_key_id` and `sse_customer_key` are mutually exclusive; only `AES256` **[src]**. |

**Environment variables.** Any optional argument can be set as `OCI_<argument>` (highest priority) or `<argument>` with no prefix. The documented example is `export OCI_region=us-ashburn-1`. The names are the lowercase argument names and are case-sensitive: I confirmed `OCI_tenancy_ocid` works and `OCI_TENANCY_OCID` (the `oracle/oci` provider's spelling) does **not** [run]. Required `bucket` and `namespace` cannot be set this way ([doc], reproduced [run]; open bug [#37277](https://github.com/hashicorp/terraform/issues/37277), where the Oracle maintainer's workaround is `-backend-config`).

**Backend blocks cannot reference variables, locals or data sources** ([Backend configuration](https://developer.hashicorp.com/terraform/language/backend)) **[doc]**. Use partial configuration (`-backend-config=PATH` or `KEY=VALUE`) for anything dynamic. Values passed in config or `-backend-config` may be persisted in `.terraform/` and plan files, so keep secrets in environment variables.

## 3. Authentication and what suits GitHub Actions

How credentials resolve **[src]** (`auth.go`): the SDK uses a composite provider that consults, in order, the auth-specific provider, then the backend's own arguments and `OCI_<arg>` env vars, then the OCI config file (`~/.oci/config`, profile from `config_file_profile` else `DEFAULT`), then the SDK's `TF_VAR_*` environment provider (`TF_VAR_tenancy_ocid`, `TF_VAR_user_ocid`, `TF_VAR_fingerprint`, `TF_VAR_private_key_path`, `TF_VAR_region`; [SDK source](https://github.com/oracle/oci-go-sdk/blob/master/common/client.go)). The config file location is `$HOME/.oci/config`, falling back to `OCI_CONFIG_FILE`. Note the TF_VAR path reads a key **file** only, not inline key text.

Tested [run] against Terraform v1.16.4 with fake credentials; success means the request reached Object Storage and was rejected with a 401 (the credentials parsed):

| Setup | Result |
|---|---|
| No credentials, empty HOME | `did not find a proper configuration for tenancy` |
| `TF_VAR_tenancy_ocid/user_ocid/fingerprint/private_key_path` | accepted |
| `OCI_tenancy_ocid/user_ocid/fingerprint` + `OCI_private_key` (inline PEM) | accepted |
| `OCI_TENANCY_OCID/...` (uppercase, provider style) | rejected |
| `auth = "OKEWorkloadIdentity"` | rejected, `Invalid authentication method` |
| `bucket`/`namespace` via `OCI_bucket`/`OCI_namespace` | rejected, `Missing Required Value` |

Options for GitHub Actions:

1. **API key via environment variables (recommended).** Create a dedicated IAM user for Terraform with an API key; store tenancy OCID, user OCID, fingerprint and private key as GitHub secrets. In the job, write the key to `$RUNNER_TEMP` and set the five `TF_VAR_*` variables. One set of variables then serves both the backend and the `oracle/oci` provider (the provider documents `TF_VAR_*` and uppercase `OCI_*` names, [Terraform provider configuration](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/terraformproviderconfiguration.htm) **[doc]**). Use `TF_VAR_*` rather than the backend-only lowercase `OCI_*` spelling so the two never diverge.
2. **API key via a config file.** Works (`config_file_profile`), but the runner has no `~/.oci/config`; you would have to template one from secrets, which adds nothing over option 1.
3. **Explicit arguments in the `backend` block.** Possible for the non-secret OCIDs, but secrets in config can be written to `.terraform/` and plan files **[doc]**. Keep the private key out of the block.
4. **`InstancePrincipal`.** Needs a compute instance in OCI. Only applicable if the apply runs on a self-hosted runner on an OCI VM (for example the free VPS). Not for GitHub-hosted runners.
5. **`SecurityToken`.** Needs a session-token profile and the provider docs say the token "expires after one hour" ([provider config](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/terraformproviderconfiguration.htm)) **[doc]**. Poor fit for unattended CI.
6. **`ResourcePrincipal` / `OKEWorkloadIdentity`.** For OCI Functions and OKE workloads. `OKEWorkloadIdentity` is currently rejected by the backend's own validation (above); a fix is open as [#37844](https://github.com/hashicorp/terraform/pull/37844) **[issue]**. Irrelevant to GitHub Actions anyway.
7. **Keyless OIDC from GitHub.** OCI IAM identity domains can exchange an external JWT for a short-lived OCI token (identity propagation trust, [Token Exchange Grant Type](https://docs.oracle.com/en-us/iaas/Content/Identity/api-getstarted/json_web_token_exchange.htm) **[doc]**), and community examples show GitHub Actions using it. But none of the backend's `auth` modes consumes such a token directly, and I did not find Oracle documentation of a supported GitHub OIDC flow for this backend. Treat as a possible later hardening step, not a baseline.

Least privilege for the state bucket **[doc]**: [Object Storage policy reference](https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/objectstoragepolicyreference.htm) maps API calls to permissions. The backend issues `ListObjects`, `HeadObject`, `GetObject`, `PutObject` (lock and state), `DeleteObject` (unlock), multipart calls for large state, and, on workspace delete, `ListObjectVersions` and `DeleteObject` with a version ID **[src]**. Mapped to permissions:

- `OBJECT_INSPECT` (list, head), `OBJECT_READ` (get), `OBJECT_CREATE` (new object, including the lock file), `OBJECT_DELETE` (unlock).
- `OBJECT_OVERWRITE` is required when `PutObject` targets an existing object, which is every state write after the first. HashiCorp's permission list does not include it (see Uncertainties), but `manage objects` covers it.
- `OBJECT_VERSION_DELETE` is needed only to delete workspaces (permanently removes all versions).
- A CMK-encrypted bucket adds key permissions on the Object Storage service principal, not on the user (below).

A simple, sufficient policy is `Allow group <tf-group> to manage objects in compartment <root or state compartment> where target.bucket.name='<bucket>'`. HashiCorp says to grant these "on the specific bucket being used" ([backend page](https://developer.hashicorp.com/terraform/language/backend/oci)). Oracle notes `target.bucket.name` matching is case-insensitive, so avoid bucket names differing only by case.

A read-only identity can serve PR plans if those use `-lock=false`, because even `plan` writes the lock object (below).

## 4. State locking

- Mechanism **[src]** (`client.go`) and **[doc]**: `Lock` does `PutObject` of a JSON `LockInfo` to `<state path>.lock` with header `If-None-Match: *`, so the write fails if the object exists. On failure it reads the existing lock and returns it in the error. `Unlock` reads the lock, checks the ID matches, then `DeleteObject` with `If-Match: <etag>`.
- Object names: default workspace `terraform.tfstate.lock`; other workspaces `tf-state-env/<ws>/terraform.tfstate.lock` ([backend page example](https://developer.hashicorp.com/terraform/language/backend/oci)). A source comment says `.tflock`, but the code uses `.lock` **[src]**, matching the docs.
- Locking is always attempted for `plan`, `apply` and `destroy` (docs); `-lock=false` and `-lock-timeout` are the standard Terraform flags.
- A killed run can leave a stale lock; clear it with `terraform force-unlock <ID>`. The ID is in the error message.
- No separate lock table or service is needed, unlike the S3 backend's DynamoDB option. Requests retry up to 5 times on 429/500/503 and network errors, with a 10-minute overall cap **[src]**.
- With versioning enabled, each lock and unlock leaves object versions and delete markers behind (see Uncertainties on whether re-locking works cleanly after a delete marker).

## 5. Workspaces

Supported **[doc]**. The default workspace is stored at exactly `key`. Others go to `<workspace_key_prefix>/<name>/<key>` with the default prefix `tf-state-env`, e.g. `tf-state-env/development/path/to/my/key`. Workspaces are discovered by listing the prefix **[src]** (needs `OBJECT_INSPECT`). Deleting a workspace deletes **all versions** of its state object and any object sharing its name prefix **[src]**, so it is irreversible. The default workspace cannot be deleted.

For this project (one environment, per the map's out-of-scope list), workspaces are unlikely to be needed. Separate `key` values per configuration are simpler and what Oracle suggests when sharing a bucket.

## 6. Bucket requirements and recommendations

Create the bucket with the `oci_objectstorage_bucket` resource ([provider docs](https://github.com/oracle/terraform-provider-oci/blob/master/website/docs/r/objectstorage_bucket.html.markdown)).

- **Versioning: enable it.** HashiCorp's page carries an explicit warning to enable bucket versioning so state can be recovered after accidental deletion or human error **[doc]**. `versioning = "Enabled"` at creation. Per [Using Versioning](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/usingversioning.htm) **[doc]**: it cannot be turned off, only **Suspended**; with it on, an overwrite keeps the old object as a previous version and a delete without a version ID creates a delete marker; all versions are billed.
- **Retention rules: don't.** Oracle: "You can't add retention rules to a bucket that has versioning enabled" and vice versa (same page) **[doc]**. Retention rules are designed to block modification or deletion for a period; I infer that they would also block Terraform's own state rewrites, so they are a poor fit even without that restriction (inference, not verified).
- **Version growth: lifecycle policy.** Oracle recommends lifecycle management to control version growth and cost, and lifecycle rules can delete previous versions (permanently) [same page]. A rule deleting previous versions after N days (and delete markers) bounds storage. State files are small, so cost is minor; Always Free object storage is 20 GB combined (or 10 GB per tier on a paid account) and 50,000 API requests a month ([Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)) **[doc]**. Each Terraform run makes on the order of ten requests (my estimate from the source), so the request quota is not a concern for a single-environment repo.
- **Private: default.** `access_type` defaults to `NoPublicAccess` ([provider docs](https://github.com/oracle/terraform-provider-oci/blob/master/website/docs/r/objectstorage_bucket.html.markdown)) **[doc]**. State files can contain secrets. Set it explicitly anyway so a drift is visible.
- **Encryption.** Always on, AES-256, with a per-object data key wrapped by a bucket master key; Oracle manages the master key by default ([Object Storage Data Encryption](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/encryption.htm)) **[doc]**. Options:
  - *Oracle-managed (default).* No setup.
  - *Customer-managed Vault key on the bucket* (`kms_key_id` on the bucket resource; optionally `is_bucket_key_enabled = true` to reduce KMS calls). Needs a Vault and key. Software-protected keys are free; virtual private vaults are not Always Free ([Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)) **[doc]**. Policies ([Common policies](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm)) **[doc]**: the service needs `Allow service objectstorage-<region> to use keys in compartment <c> where target.key.id = '<key OCID>'`, and the user assigning the key needs `use key-delegate` on that key. With this set on the bucket the backend's own `kms_key_id` is redundant (it only adds a per-object header on state uploads **[src]**; the lock object does not set it).
  - *SSE-C (`sse_customer_key`).* Not recommended: Oracle does not store the key, so losing it loses the state, and it is also written to config/`.terraform` if passed as an argument.
- **Ordering with a CMK in the same stack.** The key, the service policy and the user policy must exist before the bucket references the key; add explicit `depends_on` for the policy.
- **Bucket name.** Must match the backend's stricter regex (no periods). Bucket names are unique within the tenancy namespace; `bucket_scope` can make them region-unique.
- **Protect from deletion.** `lifecycle { prevent_destroy = true }` on the bucket resource (standard Terraform).

## 7. Bootstrap and migration when Terraform creates the bucket

The backend cannot be used before its bucket exists, and the `backend` block cannot reference the bucket resource. Standard flow (Terraform CLI behavior from [`init`](https://developer.hashicorp.com/terraform/cli/commands/init) and [Backend configuration](https://developer.hashicorp.com/terraform/language/backend) **[doc]**; not run end to end against a live tenancy):

1. **Apply with local state.** Config has no `backend` block. It creates the bucket (versioning, private, optional CMK, `prevent_destroy`), plus any key and policy it needs. Use bootstrap credentials (the ticket #17 decision).
2. **Add a partial `backend "oci" {}` block**, e.g. with `key` and `region` in the file and `bucket` and `namespace` in a `backend.hcl` passed with `-backend-config=backend.hcl`. The namespace is not secret (it is the data source `oci_objectstorage_namespace`'s output); copy it in once, since variables are not allowed in the block.
3. **Run `terraform init -migrate-state -backend-config=backend.hcl`.** `-migrate-state` copies the existing state to the new backend; re-running `init` on an already-initialized directory requires it or `-reconfigure`. `-reconfigure` would **not** migrate, so don't use it here. `-force-copy` suppresses the confirmation prompts and implies `-migrate-state`; use it only in scripts.
4. **Verify**: `terraform state list`, check the object exists in the bucket (with its first version), then remove the stale local `terraform.tfstate*` files. Back up `terraform.tfstate` before migrating (HashiCorp recommends it).
5. **Hand over credentials to the Terraform identity** (CI) and run `terraform init` there without `-migrate-state`.

Gotchas specific to a self-hosted bucket:

- The bucket's own state lives in the bucket. Replacing or destroying the bucket requires migrating state back to local first (remove the block, `init -migrate-state`). OCI also normally requires a bucket to be empty before deletion, and a versioned bucket must have all versions removed (general OCI behavior, not re-verified here), so `prevent_destroy` is the real safeguard.
- Changing the bucket's `kms_key_id` later only affects new objects; existing ones keep decrypting with the key version used when written ([encryption doc](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/encryption.htm)) **[doc]**.
- Don't enable a bucket Replication policy expecting versions to follow: replication does not replicate previous versions ([versioning doc](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/usingversioning.htm)) **[doc]**.

## 8. Known limitations and gotchas

1. `bucket` and `namespace` cannot come from env vars ([#37277](https://github.com/hashicorp/terraform/issues/37277), open since 2025-06-27) [doc, issue, run]. Use `-backend-config`.
2. Env var names are lowercase and case-sensitive (`OCI_region`, not `OCI_REGION`), unlike the provider [run]. Standardise on `TF_VAR_*` in CI.
3. `OKEWorkloadIdentity` is documented but rejected by the backend's validation ([#37844](https://github.com/hashicorp/terraform/pull/37844), open) [run, issue].
4. Bucket-name regex excludes periods, which OCI allows [src].
5. The doc's permission list omits `OBJECT_OVERWRITE` and `OBJECT_VERSION_DELETE`, both needed in practice (see section 3) [doc, src].
6. `terraform plan` takes the lock too, so a plan-only identity needs write access unless it uses `-lock=false`.
7. Stale locks need `terraform force-unlock`. The lock object name is `<key>.lock`.
8. Workspace deletion removes all object versions permanently [src].
9. The HashiCorp page does not describe each `auth` mode; the only deeper reference is the source and the provider's authentication docs.
10. The SDK pinned in 1.16.4 is v65.89.1; bumps are open ([#39359](https://github.com/hashicorp/terraform/pull/39359)) [issue]. Not a concern, but a reason to pin Terraform in CI.

## 9. Comparison with the old S3-compatible approach

| | Native `oci` backend | S3 backend against the OCI S3 Compatibility API |
|---|---|---|
| Status | Recommended by Oracle; maintained by Oracle engineers | Oracle: "deprecated. Only use this method if you're not able to upgrade to Terraform version v1.12.0 or greater" ([Oracle docs](https://docs.oracle.com/en-us/iaas/Content/dev/terraform/object-storage-state.htm)). HashiCorp tests only against Amazon S3 and "cannot offer any guarantees" for alternates ([S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)) |
| Min Terraform | 1.12.0 | Any, but needs six `skip_*`/`use_path_style` flags plus an `endpoints.s3` URL on 1.6.4+ |
| Credentials | Normal OCI API key, same as the provider | Separate **Customer Secret Key** (access key and secret key pair) held in AWS-style env vars or `~/.aws/credentials` |
| Locking | Native `<key>.lock` via `If-None-Match: *` | S3 `use_lockfile` writes `<key>.tflock` using S3 conditional writes; reported broken against OCI on 1.11.2 ([#36742](https://github.com/hashicorp/terraform/issues/36742), closed as a duplicate of #36704) and a PutObject checksum failure reported on 1.11.3 ([#36809](https://github.com/hashicorp/terraform/issues/36809)). DynamoDB locking does not exist on OCI |
| Workspaces | `tf-state-env/<ws>/<key>` | `env:/<ws>/<key>` (S3 default prefix `env:`) |
| Encryption | `kms_key_id` or bucket CMK; SSE-C | SSE-C or bucket CMK; S3 backend KMS arguments do not map to OCI |
| Maturity | Young (about 17 months old), a few open issues | Long-used, but a compatibility workaround |

Migrating an existing S3-compatible state to the native backend is a normal backend change: swap the block and run `terraform init -migrate-state`. The workspace prefix and lock suffix differ, so non-default workspaces are copied to new paths rather than reused in place. This project is greenfield (no existing state), so the old approach has no migration cost either way; the only reason to pick it would be a Terraform version below 1.12, which is not the case (local v1.16.4).

## 10. Recommendation for this project

- Use the native `oci` backend; pin Terraform to `>= 1.12` (in practice the current 1.16 line).
- State bucket in the root compartment: `versioning = "Enabled"`, `access_type = "NoPublicAccess"`, `prevent_destroy`, lifecycle rule deleting previous versions after a bounded period, no retention rules. Optional customer-managed software key if a Vault is already in the stack; otherwise Oracle-managed encryption is acceptable for a first version.
- CI: GitHub secrets for a dedicated Terraform user's API key; `TF_VAR_*` variables; `bucket`/`namespace`/`key`/`region` in a committed `backend.hcl` or `-backend-config` arguments.
- Bootstrap exactly as in section 7, scripted where possible.

## Uncertainties and things to verify

1. **Versioning plus lock re-creation.** I did not verify that `PutObject` with `If-None-Match: *` succeeds when the lock object's latest version is a delete marker (which is what an unlock produces in a versioned bucket). It almost certainly works, since the object is logically absent, but a first live `apply` followed by a second should be treated as the test. If it fails, locks would be stuck after the first run.
2. **IAM permission set.** HashiCorp lists `OBJECT_INSPECT/CREATE/DELETE/READ` only. Oracle's policy reference requires `OBJECT_OVERWRITE` for overwrites and `OBJECT_VERSION_DELETE` for version deletes. I am relying on Oracle's reference; the exact minimum should be confirmed with a least-privilege test policy.
3. **Key disable/deletion.** I could not find the Oracle statement of what happens to bucket objects if the CMK is disabled or scheduled for deletion. I expect the state to become unreadable until the key is restored, but this is unverified; treat CMK deletion as dangerous.
4. **`TF_VAR_*` fallback is SDK behavior, not a documented backend feature.** It worked in my run (the request reached the service), but it could change with an SDK bump. A fully documented alternative is the lowercase `OCI_*` names, at the cost of diverging from the provider.
5. **No live end-to-end test.** I had no real tenancy, so lock, workspace, versioning and migration behavior come from source, documentation and issues, not an actual run. The init-time credential and validation results in section 3 are real runs, but only up to a 401.
6. **OIDC from GitHub.** Whether a supported keyless path exists (token exchange then a session-token profile) was not established from primary sources; see section 3, option 7.
7. **OCI bucket deletion preconditions** (must be empty, all versions removed) are stated from general knowledge, not a page I fetched.
8. **Retention-rule semantics** (that they would block state rewrites) are my inference; the cited Oracle page only establishes that retention rules and versioning are mutually exclusive.

## Sources

- Terraform backend page: <https://developer.hashicorp.com/terraform/language/backend/oci>
- Terraform backend configuration and `init`: <https://developer.hashicorp.com/terraform/language/backend>, <https://developer.hashicorp.com/terraform/cli/commands/init>
- S3 backend (for comparison): <https://developer.hashicorp.com/terraform/language/backend/s3>
- Terraform changelog (v1.12): <https://github.com/hashicorp/terraform/blob/v1.12/CHANGELOG.md>
- Backend source: <https://github.com/hashicorp/terraform/tree/main/internal/backend/remote-state/oci>
- Implementation PR and issues: <https://github.com/hashicorp/terraform/pull/36872>, <https://github.com/hashicorp/terraform/issues/34465>, <https://github.com/hashicorp/terraform/issues/37277>, <https://github.com/hashicorp/terraform/pull/37844>, <https://github.com/hashicorp/terraform/issues/36742>, <https://github.com/hashicorp/terraform/issues/36809>
- OCI Go SDK credential providers: <https://github.com/oracle/oci-go-sdk/blob/master/common/client.go>
- Oracle: Using Object Storage for State Files: <https://docs.oracle.com/en-us/iaas/Content/dev/terraform/object-storage-state.htm>
- Oracle: Terraform provider configuration: <https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/terraformproviderconfiguration.htm>
- Oracle: Object Storage versioning, encryption, policy reference: <https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/usingversioning.htm>, <https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/encryption.htm>, <https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/objectstoragepolicyreference.htm>
- Oracle: Common policies (key use by Object Storage): <https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm>
- Oracle: Always Free resources: <https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm>
- Oracle: identity token exchange: <https://docs.oracle.com/en-us/iaas/Content/Identity/api-getstarted/json_web_token_exchange.htm>
- OCI provider bucket resource: <https://github.com/oracle/terraform-provider-oci/blob/master/website/docs/r/objectstorage_bucket.html.markdown>
