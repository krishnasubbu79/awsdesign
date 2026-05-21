# Implementation Guide 01: Account Boundary and SCP Envelope

## Purpose

This implementation guide defines how to roll out the SCP and account-boundary controls for the `vault-dev` and `vault-prod` Data Bunker accounts.

It is based on:

1. `stdb-backup-lag-hld.md`
2. `lld-01-account-boundary-scp-envelope.md`

This guide is rollout-focused. It does not try to finalize every SCP statement upfront. Where exact action lists still need validation, this guide captures sequencing, dependencies, validation, and rollback expectations.

## Scope

1. Root-level SCP change for external RAM sharing.
2. Vault-OU SCP rollout for `Vault-Dev-OU` and `Vault-Prod-OU`.
3. Account placement and exclusion prerequisites.
4. Validation and rollback approach.

## Out of Scope

1. KMS key policy implementation.
2. CloudTrail / Config / GuardDuty detailed resource creation.
3. Pull orchestration implementation.
4. Restore runbook implementation.
5. Final least-privilege IAM hardening.

## Implementation Preconditions

Before SCP rollout:

1. `Vault-Dev-OU` and `Vault-Prod-OU` exist directly under the Prod Org root.
2. The `vault-dev` and `vault-prod` accounts exist and are placed in the correct OUs.
3. These OUs and accounts are excluded from the normal Prod account-factory / baseline targeting.
4. The STDB operating roles are defined as the intended in-account operating path:
   `STDBBootstrapRole`, `STDBAdminRole`, and the SSM Automation execution role.
5. The approved Region set is locked as:
   `us-east-1`, `ap-south-1`, `ap-south-2`, `ap-northeast-2`, `ap-southeast-1`, `ap-southeast-3`, `ap-east-1`, `eu-west-1`, and `eu-west-2`.

## Rollout Strategy

Use a phased rollout:

1. Root SCP exception first.
2. Vault-OU SCPs in a light but functional form.
3. Validation of backup, KMS, RAM, and governance service paths.
4. Incremental tightening after working behavior is proven.

Apply the same SCP intent to both `Vault-Dev-OU` and `Vault-Prod-OU`.

## SCP Set

### 1. Root SCP Change: External RAM Sharing Exception

Target:

Prod Org root SCP that currently denies RAM sharing to external organizations.

Purpose:

Preserve the root-level deny for normal Prod accounts while allowing only `vault-dev` and `vault-prod` to perform restore-time external RAM sharing.

Implementation direction:

1. Modify the existing root-level deny, do not create a broad parallel allow.
2. Add an account-scoped exception for only the `vault-dev` and `vault-prod` account IDs.
3. Keep the exception scoped to the RAM share lifecycle actions required for just-in-time restore sharing.
4. Do not use a role-scoped exception in the root SCP.

Validation:

1. Confirm a normal Prod account still cannot share externally through RAM.
2. Confirm the two Data Bunker accounts are no longer blocked by the root-level deny.
3. Confirm restore-time sharing is still governed by lower-level SCP and IAM controls.

Rollback:

1. Remove the account-scoped exception from the root SCP.
2. Confirm the external RAM-sharing deny again applies universally.

### 2. Vault-OU SCP: Region Boundary

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Restrict Data Bunker activity to the approved nine Regions while preserving required global/control-plane behavior.

Implementation direction:

1. Use a deny-outside-approved-regions model.
2. Exempt the initial global/control-plane service set:
   `iam:*`, `sts:*`, and `organizations:*`.
3. Add `support:*` only if AWS Support access from the Data Bunker accounts is required.
4. Keep AWS Backup, KMS, RAM, SSM, Config, GuardDuty, and CloudTrail Region-bound unless testing proves a specific exemption is needed.

Validation:

1. Confirm the approved Regions work for AWS Backup, KMS, RAM, SSM, Config, GuardDuty, and CloudTrail.
2. Confirm an unapproved Region is blocked.
3. Confirm IAM, STS, and Organizations flows are not broken.

Rollback:

1. Detach or relax the Region SCP.
2. Re-test approved operational paths before proceeding to the next SCP stage.

### 3. Vault-OU SCP: Workload Prevention

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Prevent the Data Bunker accounts from turning into general-purpose workload accounts.

Implementation direction:

1. Deny create/mutate/provisioning actions for network resources:
   VPCs, subnets, route tables, internet gateways, NAT gateways, transit gateways, VPC endpoints, load balancers, security groups, and network ACLs.
2. Deny create/mutate/provisioning actions for general compute platforms:
   EC2, Auto Scaling, ECS, EKS, EMR, Batch, and similar platforms.
3. Deny create/mutate/provisioning actions for database and application data platforms:
   RDS, Aurora, DynamoDB, ElastiCache, Redshift, OpenSearch, Neptune, DocumentDB, and similar services.
4. Do not SCP-deny Lambda at this stage.
5. Treat Lambda as tolerated but not part of the default operating model.
6. Preserve SSM Automation as the preferred execution path.

Validation:

1. Confirm blocked resource families cannot be provisioned.
2. Confirm SSM Automation and approved control-plane services still function.
3. Confirm Lambda is not blocked if needed later.

Rollback:

1. Remove or narrow the workload-prevention SCP.
2. Re-test AWS control-plane services after rollback.

### 4. Vault-OU SCP: RAM Guardrail

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Prevent broad use of the root-level RAM exception while preserving just-in-time restore sharing.

Implementation direction:

1. Deny RAM share lifecycle actions unless the caller is an approved STDB operating path.
2. Treat the approved STDB operating paths as:
   `STDBBootstrapRole`, `STDBAdminRole`, and approved SSM Automation execution where needed.
3. Cover create, update, associate, disassociate, and delete operations in the restore-time share lifecycle.
4. Do not try to encode recovery account IDs in this SCP.
5. Leave final recovery-account validation to IAM and approved runbook logic.

Validation:

1. Confirm non-approved principals in the Data Bunker accounts cannot administer RAM shares.
2. Confirm approved STDB operating paths can complete the intended RAM workflow.
3. Confirm the root-level exception is not usable broadly by unintended principals.

Rollback:

1. Relax the vault-OU RAM guardrail while keeping the root account-scoped exception in place if needed for recovery testing.
2. If broader rollback is needed, also revert the root SCP exception.

### 5. Vault-OU SCP: IAM / Restricted Administration

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Preserve the approved STDB operating paths without over-tightening IAM during the initial working phase.

Implementation direction:

1. Allow the current role model to operate:
   `STDBBootstrapRole`, `STDBAdminRole`, and the SSM Automation execution role.
2. Avoid aggressive IAM deny statements during the first implementation pass.
3. Defer deeper privilege-escalation and `iam:PassRole` hardening until the backup workflow is working and development requirements are clearer.

Validation:

1. Confirm bootstrap and admin roles can perform the expected setup and validation tasks.
2. Confirm premature IAM restrictions do not block backup-path discovery.

Rollback:

1. Remove or relax IAM-related deny statements if they block implementation.

### 6. Vault-OU SCP: Backup and Vault Protection

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Protect LAG vaults, copied recovery points, access policies, and protection posture from arbitrary weakening.

Implementation direction:

1. Start with guardrail intent, not a maximally large deny list.
2. Validate the real AWS Backup workflow before finalizing deny statements for delete, policy-change, and protection-weakening actions.
3. Preserve legitimate copy, retention, restore, and approved share-adjacent operations.

Validation:

1. Confirm the working copy path into the LAG vault is not blocked.
2. Confirm intended vault administration paths work.
3. After workflow validation, test candidate deny actions against delete and policy-tamper scenarios.

Rollback:

1. Remove or narrow Backup-specific deny statements if they block the working vault flow.

### 7. Vault-OU SCP: KMS Allowance Boundary

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Avoid blocking the approved cross-account KMS path from the STDB Key Vault account.

Implementation direction:

1. Keep the SCP permissive enough for the approved cross-account KMS path.
2. Do not try to encode the full KMS trust model in the SCP before LLD 03 implementation is finalized.
3. Review later whether unapproved KMS key usage should be restricted.

Validation:

1. Confirm the Data Bunker accounts can use the STDB Key Vault account CMKs for LAG vault creation and operation.
2. Confirm AWS Backup service paths that depend on those keys are not blocked.

Rollback:

1. Remove or relax KMS-related deny logic if it blocks the approved KMS path.

### 8. Vault-OU SCP: Logging and Monitoring Protection

Target:

`Vault-Dev-OU` and `Vault-Prod-OU`

Purpose:

Protect CloudTrail, AWS Config, and approved monitoring-routing controls from tampering without breaking evidence delivery.

Implementation direction:

1. Start with anti-tamper intent only.
2. Validate exact service actions before implementing deny statements for:
   CloudTrail control changes, Config recorder/delivery changes, and EventBridge-based monitoring-routing changes.
3. Preserve CloudTrail delivery, Config delivery, GuardDuty publishing, EventBridge routing, and approved SSM Automation workflows.

Validation:

1. Confirm CloudTrail, Config, GuardDuty, and EventBridge paths continue to work.
2. Confirm approved governance roles can manage these controls.
3. After implementation validation, test tamper-path denies carefully.

Rollback:

1. Remove or narrow logging/monitoring deny statements if they interfere with evidence collection or routing.

## Deployment Sequence

Recommended order:

1. Confirm OU placement and account IDs.
2. Apply the root SCP RAM exception.
3. Apply the Region SCP.
4. Apply workload-prevention SCP.
5. Apply the RAM guardrail SCP.
6. Apply the light IAM/restricted-administration SCP.
7. Validate KMS, AWS Backup, CloudTrail, Config, and RAM basic paths.
8. Only then begin incremental hardening for backup protection and logging/monitoring anti-tamper controls.

## Validation Checklist

Minimum validation after first rollout:

1. `vault-dev` and `vault-prod` are in the correct OUs.
2. Unapproved Regions are blocked.
3. Approved Regions work for AWS Backup, KMS, CloudTrail, Config, GuardDuty, RAM, and SSM.
4. Network/compute/database workload creation is blocked.
5. Lambda is not blocked.
6. Root-level external RAM sharing remains denied for normal Prod accounts.
7. Data Bunker RAM sharing is only usable through approved STDB operating paths.
8. STDBBootstrapRole and STDBAdminRole remain usable.
9. Cross-account KMS use from the STDB Key Vault account is not blocked.
10. CloudTrail and Config evidence paths are not broken.

## Rollback Principles

If a rollout step blocks the approved operating path:

1. Roll back the most recently added SCP first.
2. Prefer narrowing or detaching the OU-level SCP before changing the root SCP.
3. Touch the root SCP again only when the problem is specifically tied to the external RAM-sharing deny exception.
4. Re-test STDB operating roles, KMS path, AWS Backup copy path, and governance evidence path after each rollback.

## Follow-on Work

This guide intentionally leaves some hardening for later:

1. LLD 03 / implementation for exact KMS trust and usage path.
2. LLD 05 for pull orchestration implementation.
3. LLD 06 for restore-time RAM share runbooks and lifecycle cleanup.
4. Later IAM tightening once the working backup path is proven.
5. Later action-level deny refinement for AWS Backup, CloudTrail, Config, and EventBridge anti-tamper controls.
