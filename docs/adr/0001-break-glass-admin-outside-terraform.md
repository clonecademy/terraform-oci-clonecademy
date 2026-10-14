# The break-glass admin stays outside Terraform

The tenancy repo is the single source of truth for the clonecademy tenancy, including IAM. One exception is deliberate: the break-glass admin (`karthikgappiah@gmail.com`, MFA-protected), the default `Administrators` group and the default `Tenant Admin Policy` are never managed by Terraform. A bad apply, a compromised Terraform identity or a broken state file must never be able to remove the last way back in. All other human access lives in Terraform-managed groups with narrower policies.
