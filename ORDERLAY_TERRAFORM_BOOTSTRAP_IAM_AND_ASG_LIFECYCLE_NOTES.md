# Concept Note: Bootstrapping a New Project's Terraform (orderlay), the IAM Permission Journey, and ASG Cost-Saving Gotchas

> **Audience:** Myself (and anyone else) doing this again for a *third* project, or hitting the same errors on orderlay/agentcis a second time. Written so I never have to re-debug any of this from scratch.
> **Everything here is from a live debugging session on 2026-09-09 → 2026-09-10**, spinning up `orderlay` staging by copying the `agentcis` Terraform project (`GH-infra-and-k8s-charts-central/terraform-iaac/`), plus a parallel IAM-permission debugging pass on `agentcis-development`.
> **Companion reading:** `note/TERRAFORM_SERVER_CREATION_AND_NAMING_CONCEPT_NOTES.md` for how the servers themselves get created and named once Terraform can actually run. This note is about everything that has to be true *before* `terraform apply` succeeds at all on a new AWS account.

---

## 1. TL;DR

1. Copying `project/agentcis/` to `project/orderlay/` to bootstrap a new project is the right pattern, but **6 things must change per environment**, not just `vpc-id` — see the checklist in §2. Missing any one of them fails at a different, confusing point later.
2. `ssh_key_name` is an **AWS EC2 key pair name**, not a file on your laptop — it's scoped per AWS account *and* per region, so a value copied from `agentcis` (a different account, different region) will never exist in `orderlay`'s account. You must create/import a new one there. See §3.
3. A brand-new IAM deployer user needs a *lot* of permissions to run this codebase — IAM (roles/policies/instance profiles), EC2, Auto Scaling, Route53. The full, current, working policy is in §7 — copy it wholesale for the next new project instead of rediscovering each permission one error at a time.
4. **Two AWS-specific traps cost the most time and aren't about missing permissions at all:**
   - Inline user policies are capped at **2,048 non-whitespace characters** — this policy doesn't fit. Must be a **customer-managed policy** (6,144-char cap) attached to the user instead. See §6.1.
   - The very first Auto Scaling Group ever created in a *new* AWS account needs `iam:CreateServiceLinkedRole` (one-time only, after which it's never needed again in that account). See §6.2.
   - A brand-new IAM role/instance profile used **immediately** by an ASG can fail its first launch with `Authentication Failure. Launching EC2 instance failed.` — this is IAM propagation delay, not a permissions bug. Just retry. See §6.3.
5. **Stopping an EC2 instance that belongs to an Auto Scaling Group does not save cost** — the ASG treats a stopped instance as unhealthy and replaces it. To actually pause a worker (holidays, off-hours), scale the ASG itself to 0, ideally via AWS **Scheduled Actions**, not by stopping instances. See §8.
6. Editing a local `.json` policy file does **nothing** by itself — it has to be pushed to AWS (console or `aws iam create-policy-version`) by someone with IAM rights on that specific account. Every "I already fixed it" moment where the error repeated identically was this. See §5's recurring pattern.
7. **As of 2026-09-10, only one policy file is actually in use: `/home/alija/TerraformInfrastructurePolicy.json`**, attached (as a customer-managed policy) to *both* `new-devops-user` (agentcis, account `834033184010`) and `alija-orderlay` (orderlay, account `381491939487`). The other draft, `~/Downloads/orderlay-terraform-deployer-policy.json`, is **abandoned/stale** — do not use it, do not sync from it. See §7.

---

## 2. Part A — Checklist for bootstrapping a new project by copying an existing one

This is what actually needs to change when you `cp -r project/agentcis project/orderlay` (or equivalent) and adapt it for a new AWS account. Confirmed by direct experience — missing any of these fails, but each one fails at a *different* stage, which makes it feel like unrelated bugs if you don't know this list up front.

| # | What | File | Why it must change | What happens if you forget |
|---|---|---|---|---|
| 1 | `vpc-id`, `public_subnet_ids`, `private_subnet_ids` | `project/<name>/environment/<env>/terraform.tfvars` | New project = new VPC in a different account | `aws_vpc`/`aws_subnet` data source lookups fail immediately |
| 2 | `region` + `aws_profile` | same `terraform.tfvars` | Points the AWS provider at the right account/region | Either wrong-account resources, or an `AccessDenied`/profile-not-found error before anything runs |
| 3 | `domain_name` | `project/<name>/main.tf` (hardcoded, not a tfvars var) | Becomes the private Route53 zone name (`<domain_name>-<environment>.internal`) — must be unique to avoid clashing with the other project's zone | Two projects would fight over/collide on the same private DNS zone if left as `"agentcis"` |
| 4 | `ssh_key_name` | `terraform.tfvars` | EC2 key pairs are per-account **and** per-region (see §3) — a name copied from the old project's account won't exist in the new one | `InvalidKeyPair.NotFound` on apply |
| 5 | AMI IDs (`ubuntu_24_ami` in `main.tf`, `k8s_worker_v_1_32_ami`/`v_1_35_ami`) | `main.tf` + `variables.tf` defaults + `terraform.tfvars` | AMI IDs are region-scoped; an AMI ID valid in `ap-southeast-2` doesn't exist in `ap-south-1` (or any other region) | `InvalidAMIID.NotFound` |
| 6 | `backend.hcl` `key` (state path) | `environment/<env>/backend.hcl` | Keeps the new project's Terraform state file separate from the old project's, even though they share the same S3 bucket | Without changing this, `terraform init` would attach to (and potentially corrupt) the *other* project's state |

**Not on this list, and correctly left alone:** the backend `region` field in `backend.hcl` — that's the region of the **shared state bucket** (`globalyhub-terraform-centralized-bucket`, which lives in `ap-southeast-2`), not the region resources get deployed to. Don't confuse it with `terraform.tfvars`'s `region`, which *is* the deployment region and legitimately differs per project.

**A live example of AMI verification actually mattering:** when reviewing orderlay's AMIs, `k8s_worker_v_1_32_ami`/`v_1_35_ami` were both set to `ami-00cfb0dc265a942aa` — turned out to be a valid, real AMI in `ap-south-1` (a cross-region copy of agentcis's Sydney AMI, owned by the *agentcis* account `834033184010`, shared to the orderlay account). Confirmed with:
```bash
aws ec2 describe-images --image-ids ami-00cfb0dc265a942aa --profile orderlay-staging --region ap-south-1
```
Don't assume — always verify AMI existence in the *target* account/region with a command like this before relying on it, especially when an AMI ID was copy-pasted from another project.

---

## 3. Part B — What `ssh_key_name` actually is (a recurring point of confusion)

`ssh_key_name` in `terraform.tfvars` is **not** a path to a private key file on your machine. It's the *name* of an **EC2 key pair object registered in AWS**. When an instance launches, AWS injects the key pair's **public** half into `~/.ssh/authorized_keys` for the default user — you then SSH in with the matching **private** key on your laptop.

**Critical property: EC2 key pairs are scoped per AWS account *and* per region.** `agentcis-v3-stagging` exists in the agentcis account (`834033184010`) in `ap-southeast-2`. It does not exist — and never will, just by virtue of being "the same name" — in the orderlay account (`381491939487`) in `ap-south-1`, because these are two entirely separate key-pair namespaces.

Two ways to create one in a new account:
```bash
# Option 1: let AWS generate a brand-new keypair, get back a .pem to save
aws ec2 create-key-pair --key-name orderlay-staging --region ap-south-1 \
  --profile orderlay-staging --query 'KeyMaterial' --output text > ~/.ssh/orderlay-staging.pem
chmod 400 ~/.ssh/orderlay-staging.pem   # AWS/SSH refuse world/group-readable key files

# Option 2: import the public half of a key you already have
aws ec2 import-key-pair --key-name orderlay-staging \
  --public-key-material fileb://~/.ssh/id_ed25519.pub \
  --region ap-south-1 --profile orderlay-staging
```
Whatever name you register it under becomes the value of `ssh_key_name` in that project's tfvars — it does **not** need to match the old project's key name.

Separately: this is different from `primary_admin_ssh_pub_key` / `secondary_admin_ssh_pub_key` / `k8s_worker_ssh_pub_key` in the same tfvars — those are added to `authorized_keys` later, via the boot-time `user_data` script (see the companion note), not baked in by AWS at instance launch. Two different layers of SSH access, both worth keeping straight.

---

## 4. Part C — Route53 `domain_name`: it's created, not pre-existing

Quick recap since it's what started this whole session: `domain_name` in `main.tf` (e.g. `"orderlay"`) is **not** a domain that needs to exist anywhere beforehand. It flows into `infrastructure/modules/create-services/route-53/main-r53-zone.tf`:
```hcl
resource "aws_route53_zone" "main_zone_private" {
  name = "${var.domain_name}-${var.environment}.internal"
  vpc { vpc_id = var.vpc-id }
}
```
Terraform creates a brand-new **private hosted zone** (`orderlay-staging.internal`), scoped to and only resolvable inside the given VPC. It's an internal DNS namespace, not a public/registered domain — nothing to pre-provision. The `comment` field on that resource (if you see it referenced elsewhere) is purely cosmetic AWS console metadata, no functional effect.

---

## 5. Part D — The IAM permission debugging journey, in order

This is the actual sequence of `AccessDenied`/`UnauthorizedOperation` errors hit, across both `orderlay-staging` (`alija-orderlay` user) and `agentcis-development` (`new-devops-user`), and the fix for each. **Read this before doing a third project** — it saves re-discovering all of it one error at a time.

| # | Error (the action AWS denied) | Root cause | Fix |
|---|---|---|---|
| 1 | `iam:CreateRole`, `iam:CreatePolicy` | The module unconditionally creates a GitHub-Actions IAM role and a K8s IRSA role + 2 policies (`main-template.tf:10-24`) — **regardless of the `oidc_create` flag**. Only the OIDC *provider* itself is gated by that flag. | Grant IAM role/policy create/manage actions |
| 2 | `iam:ListRolePolicies` | After creating a role, the AWS provider reads back both its *inline* policies (`ListRolePolicies`) and its *attached* policies (`ListAttachedRolePolicies`) to populate state. Only had the latter granted. | Add `iam:ListRolePolicies` |
| 3 | `ec2:DescribeLaunchTemplateVersions` | Same "read-back after write" pattern — provider re-reads each launch template's version right after creating it. | Add `ec2:DescribeLaunchTemplateVersions` |
| 4 | (proactively added, not yet hit) `autoscaling:*` entirely missing | Original hand-written policy had zero Auto Scaling permissions despite the module creating ASGs | Add full ASG lifecycle actions (Create/Update/Delete/Describe/Tags/Suspend/Resume) |
| 5 | (proactively added) `iam:DetachRolePolicy` missing | Had `AttachRolePolicy` but not its counterpart — destroy needs to detach policies before deleting roles | Add `iam:DetachRolePolicy` |
| 6 | (proactively added) Route53 only had `CreateHostedZone` | Destroying an existing zone/records needs `GetHostedZone`, `DeleteHostedZone`, `ChangeResourceRecordSets`, etc. | Fill out full Route53 lifecycle actions |
| 7 | `autoscaling:DescribeScalingActivities` | Terraform polls this after `CreateAutoScalingGroup` to confirm desired capacity was actually reached ("waiting for ASG capacity satisfied") | Add it |
| 8 | `autoscaling:SetInstanceProtection` | On `destroy`, Terraform disables scale-in protection on the ASG's instances before it can terminate them | Add it |
| 9 | **Policy exceeds 2,048 non-whitespace character limit** | This is AWS's hard cap for **inline** policies on an IAM *user* (10,240 for roles). Not a real "14 permissions" limit — see §6.1. | Convert to a **customer-managed policy**, attach it to the user instead |
| 10 | `iam:CreateServiceLinkedRole` (on `AWSServiceRoleForAutoScaling`) | The **first-ever** ASG created in an AWS account triggers AWS to auto-create this service-linked role; needs explicit permission the first time | Add scoped `iam:CreateServiceLinkedRole` (condition-restricted to `autoscaling.amazonaws.com`) — see §6.2 |
| 11 | `Authentication Failure. Launching EC2 instance failed.` (not a permissions error) | IAM propagation delay — a brand-new instance profile used immediately by the ASG hadn't yet propagated to EC2's launch backend in that region | **Just retry** `terraform apply` — nothing to fix in the policy, see §6.3 |
| 12 | `autoscaling:DetachInstances` | Tried to manually detach a running worker from its ASG (to stop it without a replacement being launched) via the console | Add `autoscaling:DetachInstances` + `AttachInstances` |

**The one non-obvious meta-lesson underlying #2, #3, #7, #8, #12:** almost every permission gap after the first few wasn't about *creating* something — it was the AWS provider **reading resources back** immediately after creating/modifying them, to keep Terraform state in sync. Any time an error message says "reading X" or "waiting for Y" rather than "creating X", it's this pattern — and it means a plain list of "create" actions per resource type is never enough; you also need the matching `Describe*`/`Get*`/`List*` read actions.

---

## 6. Part E — Three AWS behaviors that aren't obvious from the error message alone

### 6.1 Inline policy 2,048-char limit vs. managed policy 6,144-char limit

AWS enforces a combined size cap on **inline** policies per IAM user: **2,048 non-whitespace characters** (10,240 for roles, 5,120 for groups — users get the smallest). This policy's action list is comfortably past that once you cover IAM + EC2 + Auto Scaling + Route53 for real (~2,500–3,300 chars depending on version).

**The fix is not to trim permissions** — it's to stop using an inline policy at all. **Customer-managed policies** (IAM → Policies → Create policy, then attach to the user) have a separate, much larger cap: **6,144 characters**, and don't count against the inline quota. This is the AWS-intended tool for exactly this situation, not a workaround.

> A "you can only add 14 permissions" comment from an account admin, encountered mid-session, turned out to actually be this exact character-limit error — miscommunicated as a permission *count* when it's really a *character* limit on inline policies specifically. Worth clarifying explicitly if this phrasing comes up again: ask "characters or count, and inline or managed?" before trying to guess.

### 6.2 First-ever ASG in an account needs `iam:CreateServiceLinkedRole`

Every AWS account needs a service-linked role, `AWSServiceRoleForAutoScaling`, before Auto Scaling can operate in it. AWS auto-creates this role the first time you ever create an ASG in that account — which needs `iam:CreateServiceLinkedRole` granted to whoever's running Terraform, **just for that first time**. `agentcis`'s account already had this role (from prior use, before this session), so it never surfaced there. `orderlay`'s account had never had an ASG before, so it hit this immediately.

Scoped grant used (don't grant this broadly — scope it to just this one service-linked role):
```json
{
  "Sid": "AllowAutoScalingServiceLinkedRole",
  "Effect": "Allow",
  "Action": "iam:CreateServiceLinkedRole",
  "Resource": "arn:aws:iam::*:role/aws-service-role/autoscaling.amazonaws.com/AWSServiceRoleForAutoScaling",
  "Condition": { "StringEquals": { "iam:AWSServiceName": "autoscaling.amazonaws.com" } }
}
```
Alternative if you'd rather not grant this at all: have an account admin run `aws iam create-service-linked-role --aws-service-name autoscaling.amazonaws.com` once, manually, ahead of time.

### 6.3 "Authentication Failure. Launching EC2 instance failed." right after creating a new IAM role

Not a permissions bug in the deployer's policy. When a launch template references an IAM instance profile that was created **moments earlier in the same `apply`**, EC2's own launch backend in that region can lag behind IAM's control-plane propagation. The ASG's first launch attempt fails with this exact message; the role/instance profile/launch template/ASG all remain correctly created. **Just re-run `terraform apply`** — by the second attempt propagation has caught up and it succeeds using the exact same (already-existing) resources. This is a one-time hiccup per role's first-ever use, not a recurring issue.

---

## 7. Part F — Which permissions the file is currently attached where, and the file that's NOT in use

**Only `/home/alija/TerraformInfrastructurePolicy.json` is live**, as a customer-managed policy attached to:
- `new-devops-user` in the agentcis account (`834033184010`) — used for `agentcis-development`
- `alija-orderlay` in the orderlay account (`381491939487`) — used for `orderlay-staging`

Both accounts share this one policy document (its resource ARNs use wildcard accounts, `arn:aws:iam::*:...`, so it's portable). **If either environment surfaces a new gap in the future, fix it here and re-push to both accounts' attached policy — don't let them diverge.**

**`~/Downloads/orderlay-terraform-deployer-policy.json` is an earlier, abandoned draft — do not use it, do not treat it as current.** It's missing most of the fixes in §5's table (no launch-template read actions, no full Auto Scaling set, no `DetachRolePolicy`, no Route53 delete/record actions). Kept around only as debugging history; safe to delete once this note exists.

**Reminder that bit us more than once:** editing the local `.json` file changes nothing in AWS by itself. Someone with IAM admin rights on that specific account has to push it — console (IAM → Policies → the policy → Edit → JSON tab → Save) or `aws iam create-policy-version --policy-arn <arn> --policy-document file://... --set-as-default`. `alija-orderlay` and `new-devops-user` themselves don't have IAM read/write rights (by design — a deploy user shouldn't be able to self-escalate), so this always has to be a separate person/session with more access.

**Current full policy content** (as of 2026-09-10, 3,255 non-whitespace chars, well under the 6,144 managed-policy limit):

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowEC2InfrastructureManagement",
			"Effect": "Allow",
			"Action": [
				"ec2:DescribeSubnets",
				"ec2:DescribeVpcs",
				"ec2:DescribeInternetGateways",
				"ec2:DescribeRouteTables",
				"ec2:DescribeVpcAttribute",
				"ec2:DescribeSecurityGroups",
				"ec2:DescribeNetworkInterfaces",
				"ec2:DescribeInstances",
				"ec2:DescribeInstanceStatus",
				"ec2:DescribeInstanceTypes",
				"ec2:DescribeInstanceAttribute",
				"ec2:DescribeInstanceCreditSpecifications",
				"ec2:DescribeVolumes",
				"ec2:DescribeVolumeStatus",
				"ec2:DescribeTags",
				"ec2:DescribeLaunchTemplates",
				"ec2:DescribeLaunchTemplateVersions",
				"ec2:DescribeImages",
				"ec2:DescribeKeyPairs",
				"ec2:CreateSecurityGroup",
				"ec2:DeleteSecurityGroup",
				"ec2:AuthorizeSecurityGroupIngress",
				"ec2:AuthorizeSecurityGroupEgress",
				"ec2:RevokeSecurityGroupIngress",
				"ec2:RevokeSecurityGroupEgress",
				"ec2:RunInstances",
				"ec2:TerminateInstances",
				"ec2:StopInstances",
				"ec2:StartInstances",
				"ec2:ModifyInstanceAttribute",
				"ec2:CreateLaunchTemplate",
				"ec2:CreateLaunchTemplateVersion",
				"ec2:ModifyLaunchTemplate",
				"ec2:DeleteLaunchTemplate",
				"ec2:DeleteLaunchTemplateVersions",
				"ec2:CreateTags",
				"ec2:DeleteTags",
				"autoscaling:CreateAutoScalingGroup",
				"autoscaling:UpdateAutoScalingGroup",
				"autoscaling:DeleteAutoScalingGroup",
				"autoscaling:DescribeAutoScalingGroups",
				"autoscaling:DescribeScalingActivities",
				"autoscaling:SetInstanceProtection",
				"autoscaling:CreateOrUpdateTags",
				"autoscaling:DeleteTags",
				"autoscaling:SuspendProcesses",
				"autoscaling:ResumeProcesses",
				"autoscaling:DetachInstances",
				"autoscaling:AttachInstances"
			],
			"Resource": "*"
		},
		{
			"Sid": "AllowTerraformIAMManagement",
			"Effect": "Allow",
			"Action": [
				"iam:CreateRole",
				"iam:GetRole",
				"iam:TagRole",
				"iam:ListRolePolicies",
				"iam:ListAttachedRolePolicies",
				"iam:ListInstanceProfilesForRole",
				"iam:DeleteRole",
				"iam:CreatePolicy",
				"iam:GetPolicy",
				"iam:GetPolicyVersion",
				"iam:ListPolicyVersions",
				"iam:CreatePolicyVersion",
				"iam:DeletePolicyVersion",
				"iam:SetDefaultPolicyVersion",
				"iam:TagPolicy",
				"iam:AttachRolePolicy",
				"iam:DetachRolePolicy",
				"iam:DeletePolicy",
				"iam:CreateInstanceProfile",
				"iam:GetInstanceProfile",
				"iam:TagInstanceProfile",
				"iam:AddRoleToInstanceProfile",
				"iam:RemoveRoleFromInstanceProfile",
				"iam:DeleteInstanceProfile",
				"iam:PassRole",
				"iam:ListAttachedUserPolicies"
			],
			"Resource": "*"
		},
		{
			"Sid": "AllowRoute53InfrastructureManagement",
			"Effect": "Allow",
			"Action": [
				"route53:CreateHostedZone",
				"route53:GetHostedZone",
				"route53:DeleteHostedZone",
				"route53:ListHostedZones",
				"route53:ListHostedZonesByVPC",
				"route53:ChangeResourceRecordSets",
				"route53:ListResourceRecordSets",
				"route53:GetChange",
				"route53:ChangeTagsForResource",
				"route53:ListTagsForResource",
				"route53:AssociateVPCWithHostedZone",
				"route53:DisassociateVPCFromHostedZone"
			],
			"Resource": "*"
		},
		{
			"Sid": "AllowAutoScalingServiceLinkedRole",
			"Effect": "Allow",
			"Action": "iam:CreateServiceLinkedRole",
			"Resource": "arn:aws:iam::*:role/aws-service-role/autoscaling.amazonaws.com/AWSServiceRoleForAutoScaling",
			"Condition": {
				"StringEquals": { "iam:AWSServiceName": "autoscaling.amazonaws.com" }
			}
		},
		{
			"Sid": "AllowGithubOidcProvider",
			"Effect": "Allow",
			"Action": [
				"iam:CreateOpenIDConnectProvider",
				"iam:GetOpenIDConnectProvider",
				"iam:DeleteOpenIDConnectProvider",
				"iam:TagOpenIDConnectProvider",
				"iam:UpdateOpenIDConnectProviderThumbprint",
				"iam:ListOpenIDConnectProviders"
			],
			"Resource": "*"
		}
	]
}
```

The last statement (`AllowGithubOidcProvider`) was added proactively — `oidc_create = false` in both environments' tfvars today, so this module is currently skipped entirely, but the moment either gets flipped to `true`, this permission set is already in place instead of triggering another round of the same debugging.

---

## 8. Part G — ASG lifecycle: why "Stop" doesn't save money, and what actually does

### 8.1 The trap

An EC2 instance that belongs to an Auto Scaling Group is **not** a free-standing server, even though it looks like one in the EC2 console. The ASG continuously enforces its `desired_capacity` (e.g. `1`). If you `Stop` an ASG-managed instance, the ASG's health check sees "not running" as unhealthy, **terminates it, and launches a replacement** to get back to the desired count — the exact opposite of the intended cost saving.

Contrast with `k8s-root-master-nginx` and `k8s-nfs-storage` — these are plain standalone `aws_instance` resources (`ec2-list` module, not ASG-backed). Stop/Start on those works completely normally.

**How to tell them apart at a glance in the AWS console:** ASG-managed instance names end in `-instance` and are launched via a Launch Template (visible under EC2 → Auto Scaling Groups); standalone ones don't have that suffix and were created directly.

### 8.2 Three ways to actually reduce cost on an ASG-managed worker

| Approach | How | Best for |
|---|---|---|
| **Scale ASG to 0** (console/CLI) | Set `Desired capacity = 0` and `Min = 0` on the ASG; reverse when needed | One-off, ad hoc pause |
| **AWS Scheduled Actions** (recommended for recurring) | EC2 → Auto Scaling Groups → your ASG → Automatic scaling → Scheduled actions. Define cron-like rules, e.g. scale to 0 every evening, back to 1 every morning; add one-off rules for specific holiday dates | Recurring off-hours/holiday shutdown, unattended |
| **Detach Instances** (console: Instance management tab → select instance → Actions → Detach, **with "decrement desired capacity" checked**) | Removes the instance from ASG management entirely — can then be stopped/started freely, forever, without the ASG replacing it | Keeping one *specific* instance around long-term without it being ASG-managed anymore — not reversible back into ASG management |

CLI for the "scale to 0" approach:
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "[orderlay-staging]-default-worker-asg" \
  --min-size 0 --desired-capacity 0 \
  --profile orderlay-staging --region ap-south-1
# ...and to bring it back:
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name "[orderlay-staging]-default-worker-asg" \
  --min-size 1 --desired-capacity 1 \
  --profile orderlay-staging --region ap-south-1
```

### 8.3 The Terraform-vs-out-of-band conflict — read before using Scheduled Actions on a Terraform-managed ASG

If Scheduled Actions (or any manual console/CLI capacity change) is used on an ASG that Terraform also manages, **the next `terraform apply` will detect the capacity as drift and silently reset it** back to whatever's in `terraform.tfvars` — undoing the schedule's scale-down without any obvious error.

`infrastructure/modules/create-services/auto-scaling-group/asg-main.tf` already anticipates this, but it's currently **commented out**:
```hcl
  # lifecycle {
  #   ignore_changes = [desired_capacity]
  # }
```
Uncommenting this tells Terraform "don't treat `desired_capacity` changes as drift" — required if Scheduled Actions is going to be used seriously/long-term on a Terraform-managed ASG. Left alone (commented) for now since it wasn't yet needed in practice — console-driven Scheduled Actions was recommended as the starting point precisely *because* it avoids touching the Terraform code.

Also worth knowing: `min_size` for the `default-worker` ASG is **hardcoded to `"1"`** in `main-template.tf` (`asg_ec2_min_capacities = ["1"]`), not driven by any tfvars variable the way `desired_capacity` is. Scaling `min` to `0` (needed for a full shutdown) can only be done out-of-band (console/CLI/Scheduled Action) — a tfvars edit has no lever for it at all currently.

---

## 9. Common mistakes (so future-me doesn't repeat them)

**Mistake 1: Assuming an edited local policy JSON file is "done."** It isn't — it has to be pushed to AWS by someone with IAM rights on that account. This exact confusion repeated across at least 3 separate errors in this session (re-hitting `iam:ListRolePolicies` after "fixing" it locally was the clearest case).

**Mistake 2: Copying `ssh_key_name`/AMI IDs/`vpc-id` from another project without checking they exist in the new account+region.** Every one of these is account/region-scoped in AWS; a value that works in `agentcis` (ap-southeast-2) is not portable to `orderlay` (ap-south-1) just because it's the same string.

**Mistake 3: Trying to fit a real-world Terraform-deployer policy into an inline user policy.** The 2,048-char inline cap is a hard AWS quota, not something to work around by trimming permissions you actually need. Go straight to a customer-managed policy for anything beyond a trivial read-only policy.

**Mistake 4: Interpreting `Authentication Failure. Launching EC2 instance failed.` as a missing permission and going hunting for one.** When it follows *immediately* after creating a brand-new IAM role/instance profile in the same apply, it's propagation delay — retry before adding anything to the policy.

**Mistake 5: Stopping an ASG-managed EC2 instance expecting it to stay off.** It won't — the ASG replaces it. Always control cost through the ASG's desired capacity, not the instance's power state.

**Mistake 6: Letting Scheduled Actions and Terraform fight over `desired_capacity` without the `ignore_changes` lifecycle rule.** Silent reversion, not an error — easy to miss until you notice the schedule "isn't working."

---

## 10. Quick reference — files touched most in this kind of work

| What | Path |
|---|---|
| The live, in-use deployer policy | `/home/alija/TerraformInfrastructurePolicy.json` — attached to both `new-devops-user` (agentcis) and `alija-orderlay` (orderlay) |
| Abandoned draft policy — do not use | `~/Downloads/orderlay-terraform-deployer-policy.json` |
| Per-environment config that must be customized per new project | `project/<name>/environment/<env>/terraform.tfvars` |
| Hardcoded per-project values not in tfvars (`domain_name`, `ubuntu_24_ami`) | `project/<name>/main.tf` |
| State backend config (bucket/key/region) | `project/<name>/environment/<env>/backend.hcl` |
| ASG module — where `min_size` is hardcoded and `ignore_changes` is commented out | `infrastructure/modules/create-services/auto-scaling-group/asg-main.tf` |
| Where IAM roles/policies get created regardless of `oidc_create` | `infrastructure/templates-agentcis/main-template.tf:10-24` |
| Route53 private zone creation | `infrastructure/modules/create-services/route-53/main-r53-zone.tf` |

---

## 11. Open / unclear items

- Whether `TerraformInfrastructurePolicy.json` should eventually be split back into two account-specific policies (agentcis vs orderlay) as each account's actual usage diverges (e.g. if orderlay never turns on `other_auto_scaling` or OIDC, it's carrying permissions it'll never use) — currently kept as one shared file deliberately, for simplicity, while both projects are still early/small.
- `project/orderlay/environment/production/terraform.tfvars` has `region = "ap-southeast-2"` while staging is `ap-south-1` — flagged during review as worth double-checking with whoever set up the production account; not yet confirmed either way.
- Whether to uncomment the `lifecycle { ignore_changes = [desired_capacity] }` block in the ASG module now, or wait until Scheduled Actions is actually adopted for real — left commented out for now.

---

*Companion reading: `note/TERRAFORM_SERVER_CREATION_AND_NAMING_CONCEPT_NOTES.md` for what happens after `terraform apply` succeeds — how each server actually gets its name at three different layers.*
