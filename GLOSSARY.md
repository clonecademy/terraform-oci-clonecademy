# clonecademy OCI tenancy

The clonecademy Oracle Cloud tenancy, managed as code from this repository, and the family of reusable modules it consumes to stay within the Always Free Tier.

## Language

### Repositories

**Tenancy repo**:
This repository (`clonecademy/terraform-oci-clonecademy`), the single source of truth for everything that exists in the clonecademy tenancy.
_Avoid_: root repo, infra repo, live repo

**Module**:
One of the reusable, independently versioned `clonecademy/terraform-oci-<name>` repositories (vcn, vps, bucket, mysql, bastion, vault, logging, notifications, budget, free). The module template and the shared workflows repository are not modules.
_Avoid_: component, package, stack

**Module template**:
The template repository (`clonecademy/terraform-oci-module-template`) from which every module repository is created.
_Avoid_: boilerplate, skeleton, starter

### What gets built

**Free stack**:
The bundle composed by the `free` module: one dedicated compartment containing exactly one each of a VCN, a VPS, a bucket, a MySQL HeatWave instance, a bastion, a vault, a log group and a free-tier budget, together using up the tenancy's Always Free allowance.
_Avoid_: free tier environment, sandbox, landing zone

**VPS**:
A single Always Free compute instance, as provisioned by the `vps` module. Use "compute instance" only where the OCI term matters.
_Avoid_: server, VM, node

**State bucket**:
The Object Storage bucket in the root compartment that holds the tenancy repo's own Terraform state.
_Avoid_: backend bucket, tfstate bucket

**Free-tier budget**:
A budget whose alerts fire on any spend above (near) zero, signalling that something has left the Always Free Tier. Applied both to a free stack and to the whole tenancy.
_Avoid_: cost guard, spending limit

**Always Free guard**:
A rule inside a module that refuses any configuration that would leave the Always Free Tier.
_Avoid_: free-tier check, limit validation

### Access

**Terraform identity**:
The dedicated, non-human OCI user whose API key the tenancy repo uses for every plan and apply; it and its least-privilege policies are managed by the tenancy repo itself.
_Avoid_: service account, CI user, bot

**Operator**:
A human whose access the tenancy repo manages, who can run and inspect a free stack but never reshape it; structural changes go through the tenancy repo.
_Avoid_: user, developer, admin

**Break-glass admin**:
The single human administrator deliberately left outside Terraform, so no apply can ever lock everyone out of the tenancy.
_Avoid_: root user, owner, super admin
