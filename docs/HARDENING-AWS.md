# AWS hardening guideline for the EKS platform

| Item | Value |
|---|---|
| Status | Draft 0.1, written 2026-10-24. A guideline for a Terraform module author. It is not the module. |
| Scope | One AWS account, `us-west-2`, one EKS cluster (Kubernetes 1.37, one `c6i.xlarge` Amazon Linux 2023 amd64 managed node), a VPC, an NLB created by ingress-nginx, Argo CD, cert-manager and Podium. See `docs/SPEC.md`. |
| Rule for conflicts | If a CIS benchmark and a DISA STIG disagree, CIS wins. This is not a government environment. Each case is recorded in section 7. |
| Honesty marker | A line marked `UNVERIFIED` was not confirmed from a source that was read. The text next to it says what would confirm it. |

## 1. How to use this document

1. Section 2 states which benchmark versions were used and which sources could not be read.
2. Section 3 lists questions only the owner can answer. Several change the design.
3. Section 4 answers "what must be installed or enabled in AWS": every service and resource, whether this project needs it, and a cost note.
4. Section 5 answers "how should each be configured": one row per control, grouped by area. Section 5.0 defines shared building blocks that later rows refer to.
5. Section 7 (Conflicts) and section 8 (Exceptions and deviations) must be read before implementing. Some controls cannot be met on this project, and they are listed rather than hidden.
6. Section 9 designs the verification and validation tool.
7. Section 10 is the acceptance checklist and section 11 lists the sources.

### Row format

Each control row has these fields: stable ID, source, Level, requirement (paraphrased), whether this project needs it, the Terraform that implements it, how to verify it, and dependencies.

- **ID** is `<SOURCE>-<BENCHMARK>-<VERSION>/<control number>`, for example `CIS-AWS-7.0.0/6.7`. IDs that do not come from a benchmark use `LOCAL/<name>` and are marked as project additions.
- **Src**: `CIS` or `STIG` or `AWS` (AWS's own EKS best-practices guide) or `LOCAL`.
- **Lvl**: the CIS Level (1 or 2) where it was readable. `?` means UNVERIFIED because the CIS document was not readable. STIG rows give no Level.
- **Needed**: `Y` required for this build; `N/A` cannot apply here (reason given); `Opt` applies but costs real money, so the owner decides; `Dev` a deliberate deviation listed in section 8 as `EX-nn`.
- **Terraform** uses AWS provider resource and argument names. Every AWS resource and argument named here was checked against the Terraform AWS provider documentation for provider v6.70.0 (latest release when checked, published 2026-10-22), unless the cell says UNVERIFIED. The exceptions are `random_password` and `kubernetes_secret`, which belong to other providers and were not checked, and the advanced event selector field names, which were checked only for `eventCategory` and `resources.type` (`readOnly` is UNVERIFIED). Anything created inside the cluster (Kubernetes objects, Helm values, Service annotations) is described as such; Terraform is not the right tool for the in-cluster items that Argo CD owns.
- **Verify** gives an AWS CLI command, a Security Hub control ID or Config rule, or a `kubectl` or `kube-bench` check. Security Hub control IDs are from the Security Hub CSPM documentation read on 2026-10-24.

## 2. Benchmarks and versions used, and what could not be read

### 2.1 Versions

| Standard | Version used for IDs | Latest seen | Date | Notes |
|---|---|---|---|---|
| CIS AWS Foundations Benchmark | 7.0.0 for numbering | 7.1.0 listed as current on the CIS AWS page (page checked 2026-10-24) | No date on any page that was readable | The 7.0.0 control numbers and Levels come from a public machine-readable copy in the Prowler repository (`cis_7.0_aws.json`), not from the CIS PDF. 7.1.0 text was not readable, so any renumbering between 7.0.0 and 7.1.0 is UNVERIFIED. A person with a CIS login should diff the two before the module is accepted. Security Hub CSPM supports up to 5.0.0 only (per its documentation), so Security Hub IDs below use the 5.0.0 mapping as a cross-check. |
| CIS Amazon EKS Benchmark | 1.8.0 for numbering (via kube-bench `eks-1.8.0`) | 2.0.0 (listed on the CIS Kubernetes page and in the CIS "June 2026 update" post, which says it supports Kubernetes 1.34 and 1.35) | 2.0.0 date not stated | The 2.0.0 text and change list were not readable. kube-bench has no `eks-2.0.0` config yet, so control numbers below follow 1.8.0 and are UNVERIFIED against 2.0.0. Kubernetes 1.37 is outside the range the 2.0.0 post states. |
| CIS Amazon Linux 2023 Benchmark | 1.0.0 | 1.0.0 | 2023-06-26 (from the README of an open-source Ansible role built on it) | Control numbers were not readable. The node OS is hardened by the Ansible role, not by this document. |
| CIS AWS Compute Services Benchmark | 2.0.0 listed | 2.0.0 | not stated | Not readable. Its scope list for earlier versions names EC2, Lambda, Batch, Lightsail and similar; whether 2.0.0 adds anything for EKS nodes is UNVERIFIED. |
| CIS AWS Storage Services Benchmark | 1.0.0 | 1.0.0 | header of a mirrored PDF reads 07-03-2024 | A mirrored PDF was readable as text. Its recommendations are all "Manual" and are mostly how-to procedures for backup, EBS, EFS, FSx and Elastic Disaster Recovery. Nothing in it adds a requirement beyond the AWS Foundations Benchmark for this build. |
| CIS AWS Database Services, End User Compute | not used | 2.0.0, 1.2.0 | | Not in the build. |
| CIS Kubernetes Benchmark | not used | 2.0.1 | | The control plane is AWS-managed, so the EKS benchmark applies instead. |
| CIS Amazon EKS Auto Mode Benchmark | not used | 1.0.0 | | The build uses a managed node group, not Auto Mode. |
| CIS Docker Benchmark | not used | 1.9.0 | | Nodes run containerd, not Docker. |
| CIS NGINX benchmark | not assessed | not checked | | ingress-nginx is configured through a ConfigMap, not an `nginx.conf` on a host. See `LOCAL/INGRESS-*` rows. |
| DISA STIG: Kubernetes | V2R6 | V2R6 | 2026-02-12 | Read through a third-party audit file (first 50 of 93 checks) and a STIG listing site. |
| DISA STIG: Amazon Linux 2023 | V1R4 | V1R4 | 2026-05-22 | 187 findings (20 CAT I, 164 CAT II, 3 CAT III). About 30 rule titles were read. |
| DISA SRG: Container Platform | V2R4 | V2R4 | 2025-09-10 | Listed only. Not read. Generic requirements that the Kubernetes STIG implements. |
| DISA STIG: NGINX | V1R1 | V1R1 | 2026-01-07 | Listed only. For an NGINX web server install, not ingress-nginx. Not read. |
| DISA SRGs: Cloud Computing Mission Owner Network and Operating System | V1R2, V1R3 | | 2024-12-20, 2025-08-13 | Listed only. Written for DoD cloud mission owners. Not applicable here. |

### 2.2 Which DISA STIGs exist for this build

Checked against a public mirror of DISA's library (435 titles) on 2026-10-24. The DISA site itself returned an empty landing page to the fetch tool.

- **There is no AWS-wide STIG** and no STIG for AWS Foundations, IAM, VPC, S3, CloudTrail or KMS in that list.
- **There is no STIG for Amazon EKS** in that list. The only EKS link found is that kube-bench ships an `eks-stig-kubernetes-v1r6` config, which runs a subset of Kubernetes STIG V1R6 checks against EKS nodes. That is a tool author's mapping, not a DISA publication.
- Relevant STIGs that exist: Kubernetes V2R6, Amazon Linux 2023 V1R4, Container Platform SRG V2R4, and NGINX V1R1. The two Cloud Computing Mission Owner SRGs exist but are DoD-specific.
- Docker Enterprise and Terraform have no STIG in that list. The Ansible items in the list are for a different product (Automation Controller).
- The mirror could be stale or incomplete. Confirm on DISA's own library before relying on "no STIG exists".

### 2.3 Sources that could not be read

- CIS benchmark PDFs (registration required): AWS Foundations 7.1.0 and 7.0.0, EKS 2.0.0 and 1.8.0, Amazon Linux 2023 1.0.0, AWS Compute Services 2.0.0. Substitutes: Prowler's JSON copy of 7.0.0 (control numbers, titles, Levels, automated/manual flags), kube-bench's YAML for EKS 1.8.0 (control numbers, titles, audit commands), Security Hub CSPM documentation (v5.0.0 and earlier mappings).
- DISA's library (`cyber.mil`): empty page. Substitute: a public STIG listing site.
- Page 2 of the Kubernetes STIG audit file (checks 51 to 93) and the full text of the AL2023 STIG: not read. Rows from them are not in this document.
- Terraform Registry pages: render with JavaScript and returned nothing. Substitute: the provider's own markdown documentation on GitHub (main branch, which matched provider release v6.70.0 when checked).
- AWS pricing pages: not read. No dollar figure in this document is from AWS pricing. Cost notes are categories (free, fixed monthly, hourly, per GB or per event). The only price in the project is the node figure in `docs/SPEC.md`, which is also unconfirmed.

## 3. Questions for the owner

1. **ingress-nginx is archived.** The `kubernetes/ingress-nginx` repository is archived on GitHub, its last release is `controller-v1.15.1` (2026-03-19), and the Kubernetes announcement says there will be no further releases, bug fixes or security fixes after March 2026. The spec builds on it. Keep it for the demo, or choose another ingress or Gateway API controller? This document keeps the NLB design neutral on the controller.
2. **Is the account part of an AWS Organization?** If not, `CIS-AWS-7.0.0/2.1.x` is not applicable and `CloudTrail`, `Config` and `Access Analyzer` run per account.
3. **Which paid detective services are wanted?** GuardDuty, Config recording, Security Hub, S3 data events in CloudTrail, interface VPC endpoints, Macie and Inspector all cost money while the demo is up. Section 4 flags each. The module should expose a boolean per service.
4. **Public API endpoint.** The cluster is operated from a workstation, so a private-only endpoint (`CIS-EKS-1.8.0/5.4.2`) would need a VPN or a tunnel. The default here is public access limited to the operator's address (`EX-01`). Acceptable?
5. **Deployer permissions.** The identity that runs Terraform probably holds an administrator policy, which fails `CIS-AWS-7.0.0/2.14` if it is attached as an AWS managed policy. Should the module author also produce a scoped deployer role?
6. **Root account controls** (`2.4`, `2.5`, `2.6`, MFA delete) cannot be set by Terraform. Who confirms them, and is a hardware key available?
7. **CIS versions.** Should the module target 7.0.0 numbering (readable here), or wait until someone with a CIS login has confirmed 7.1.0 and EKS 2.0.0 numbering?
8. **Baseline lifetime.** Should account-level services (CloudTrail, Config, Security Hub, GuardDuty, password policy, S3 account block) live in their own Terraform root that the teardown script does not destroy? This document assumes a new root, `terraform/account/`, with its own state, and that teardown leaves it alone unless told otherwise.

## 4. What needs to be installed or enabled in AWS

Cost vocabulary: **free**; **fixed** (monthly fee per resource); **hourly** (billed while it exists); **volume** (per GB or per event). None of these were price-checked. "Controls" lists the rows in section 5 that need it.

| # | Service or resource | Needed here | Cost class and note | Controls |
|---|---|---|---|---|
| 1 | IAM account password policy, account contacts, support role, no root keys, MFA on root | Y | free. Root items are manual (`EX-02`). | CIS-AWS 2.2 to 2.21 |
| 2 | IAM Access Analyzer (external access analyzer) | Y | believed free for the external access analyzer type and paid for the unused-access types (UNVERIFIED, pricing not read). Per region. | CIS-AWS 2.18 |
| 3 | CloudTrail multi-Region trail, log file validation, KMS encryption, CloudWatch Logs delivery | Y | Management events cost little; storage and CloudWatch ingestion are volume (UNVERIFIED, pricing not read). | CIS-AWS 4.1, 4.2, 4.5; 5.x |
| 4 | CloudTrail S3 data events, read and write | Opt | volume (per event). Terraform's state polling creates many events on the state bucket. Required for the L2 rows 4.8 and 4.9 if all buckets are covered. | CIS-AWS 4.8, 4.9 |
| 5 | CloudWatch Logs groups with retention and KMS: CloudTrail, EKS control plane, VPC flow logs (if sent there) | Y | volume (ingest per GB, storage per GB-month). EKS audit logs are the largest source. | CIS-EKS 2.1.x; CIS-AWS 4.7, 5.x |
| 6 | CloudWatch metric filters and alarms (14 patterns plus Organizations changes), SNS topic with an email subscription | Y (L1 rows) / Opt (L2 rows) | fixed per alarm, small. | CIS-AWS 5.1 to 5.15 |
| 7 | AWS Config configuration recorder, delivery channel, service-linked role | Opt (L2 row 4.3; also needed by Security Hub) | volume (per configuration item recorded). A cluster that churns resources records a lot. Daily recording or a resource-type list reduces it. | CIS-AWS 4.3; Security Hub |
| 8 | AWS Security Hub CSPM with the CIS standard | Opt (L2 row 5.16) | volume (per check and per finding), and needs Config. Tracks CIS 5.0.0, not 7.x. | CIS-AWS 5.16 |
| 9 | GuardDuty with EKS audit log monitoring, optional runtime monitoring and S3 protection | Opt (AWS best practice, not in the CIS AWS list) | volume. Runtime monitoring adds an agent add-on that uses node memory. | AWS rows |
| 10 | VPC flow logs | Y (L2 row 4.7) | volume. Sending to S3 is usually cheaper than CloudWatch Logs. | CIS-AWS 4.7 |
| 11 | KMS customer managed keys with rotation: EKS secrets, EBS, logs, S3 | Y | fixed per key per month plus request volume. Four keys is the clean design; merging to two is acceptable. | CIS-AWS 4.5, 4.6; CIS-EKS 5.3.1 |
| 12 | EBS default encryption, default KMS key | Y | free. | CIS-AWS 6.1.1 |
| 13 | S3 account-level Block Public Access | Y | free. | CIS-AWS 3.1.4 |
| 14 | S3 buckets: Terraform state, access-log target, CloudTrail, Config, flow logs, Ansible playbook | Y | volume, small. | CIS-AWS 3.1.x, 4.4 |
| 15 | VPC, public and private subnets in two AZs, route tables, internet gateway, single NAT gateway | Y | NAT is hourly plus per GB. | CIS-AWS 6.x; CIS-EKS 5.4.3 |
| 16 | VPC gateway endpoint for S3 | Y (cheap) | free. | CIS-AWS 6.8 |
| 17 | VPC interface endpoints (STS, EC2, SSM, SSM messages, EC2 messages, CloudWatch Logs, KMS) | Opt | hourly per endpoint per AZ plus per GB. Seven endpoints in two AZs is a large share of a demo's cost. | CIS-AWS 6.8 |
| 18 | EKS cluster with control plane logging, secrets encryption, access entries, restricted endpoint | Y | EKS control plane is hourly. | CIS-EKS 2.x, 5.x |
| 19 | EKS managed node group, launch template (IMDSv2, encrypted volume, no public IP), node IAM role | Y | The node is hourly. | CIS-AWS 6.7; CIS-EKS 3.x |
| 20 | EKS add-ons: VPC CNI (network policy on), CoreDNS, kube-proxy, EBS CSI, Pod Identity Agent | Y | free apart from node resources. | CIS-EKS 4.3.1, 5.2.1 |
| 21 | Pod Identity associations (or IRSA) for the EBS CSI driver and VPC CNI | Y | free. | CIS-EKS 5.2.1 |
| 22 | Systems Manager: managed instance, association for Ansible, document sharing blocked | Y | free for these features. Session logging to CloudWatch or S3 is volume. | AWS rows |
| 23 | AWS Load Balancer Controller, only if NLB attributes (deletion protection, access logs, security groups) are to be set by annotation | Opt (see section 5.8) | free software; uses node memory and needs an IAM role. | CIS-AWS 4.10 |
| 24 | NLB (created by the ingress Service) with security group and access logs | Y | hourly plus per LCU. | CIS-AWS 4.10, 6.3 |
| 25 | AWS Budgets alarm | Y (already in spec) | free for the first budgets. | LOCAL |
| 26 | Amazon Inspector (EC2 scanning) | Opt | volume per instance. | AWS rows |
| 27 | Amazon Macie | Opt (L2 row 3.1.3, manual) | volume per bucket and per GB inspected. Little value for a state bucket. | CIS-AWS 3.1.3 |
| 28 | AWS Organizations controls (SCPs, delegated admin) | N/A unless in an Organization | free. | CIS-AWS 2.1.x |
| 29 | ECR, RDS, EFS, ACM, Route 53, WAF | N/A | Not in the build. Their CIS rows are listed as N/A for completeness. | CIS-AWS 3.2.x, 3.3.1; CIS-EKS 5.1.x |

### Suggested module switches

`enable_guardduty`, `enable_guardduty_runtime`, `enable_config`, `config_recording_frequency`, `enable_securityhub`, `enable_s3_data_events`, `enable_interface_endpoints`, `enable_macie`, `enable_inspector`, `eks_log_types`, `eks_public_access_cidrs` (no default, must be set), `alarm_email`, `log_retention_days`, `enable_deletion_protection` (default false because of teardown, `EX-05`).

## 5. Controls and configuration

### 5.0 Shared building blocks (project additions)

Later rows refer to these by ID.

| ID | Block | Terraform | Verify |
|---|---|---|---|
| LOCAL/KMS-EKS | Key for EKS secrets envelope encryption | `aws_kms_key` (`enable_key_rotation = true`, `deletion_window_in_days = 7`, `policy` = account root plus the cluster role limited to `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey`, `kms:ListGrants`, `kms:CreateGrant`), `aws_kms_alias` | `aws kms get-key-rotation-status --key-id <id>` |
| LOCAL/KMS-EBS | Key for EBS (node root volume, PVCs, default encryption) | `aws_kms_key` with rotation. Key policy lets the node group's service-linked role (`AWSServiceRoleForAutoScaling`) and the EBS CSI role use it (`kms:CreateGrant`, `kms:GenerateDataKeyWithoutPlaintext`, `kms:Decrypt`, `kms:DescribeKey`, `kms:ReEncrypt*`). | same |
| LOCAL/KMS-LOGS | Key for CloudWatch Logs, CloudTrail, SNS | `aws_kms_key` with rotation. Key policy statements for `logs.us-west-2.amazonaws.com` (condition `kms:EncryptionContext:aws:logs:arn`) and `cloudtrail.amazonaws.com` (condition `aws:SourceArn` and `kms:EncryptionContext:aws:cloudtrail:arn`). UNVERIFIED: exact condition keys; check the CloudTrail and CloudWatch Logs KMS pages. | same |
| LOCAL/KMS-S3 | Key for the state, playbook, Config and flow log buckets | `aws_kms_key` with rotation. The state bucket must also allow the operator role. | same |
| LOCAL/BKT-LOG | S3 server access log target bucket | `aws_s3_bucket`; `aws_s3_bucket_public_access_block` (all four true); `aws_s3_bucket_server_side_encryption_configuration` with `sse_algorithm = "AES256"` (SSE-S3, because the log service cannot write to a bucket whose default encryption is SSE-KMS, per the S3 server access logging guide); `aws_s3_bucket_ownership_controls` (`BucketOwnerEnforced`); `aws_s3_bucket_policy` allowing `logging.s3.amazonaws.com` with `aws:SourceArn` and denying `aws:SecureTransport = false`; `aws_s3_bucket_lifecycle_configuration` expiring objects | `aws s3api get-bucket-encryption --bucket <b>` |
| LOCAL/SNS-ALERT | Alarm topic | `aws_sns_topic` (`kms_master_key_id` = LOCAL/KMS-LOGS alias), `aws_sns_topic_subscription` (protocol `email`) | `aws sns list-subscriptions-by-topic --topic-arn <arn>` |
| LOCAL/TF-TAGS | Provider `default_tags` carrying `name`, an environment tag and the hardening tag the SSM association targets | provider `default_tags` | `aws resourcegroupstaggingapi get-resources --tag-filters Key=<k>` |

### 5.1 Account and IAM (CIS AWS Foundations, section 2)

Source for this group: Prowler `cis_7.0_aws.json` (numbers, titles, Levels) and Security Hub CSPM mappings (control IDs). Account-level resources go in the `terraform/account/` root.

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-AWS-7.0.0/2.1.1 to 2.1.6 | CIS | 2 | Organization-level controls: centralized root access, guardrail policies, management account kept free of workloads, OU structure, delegated administrators | N/A (EX-17) unless the account is in an Organization | none | `aws organizations describe-organization` (fails with an error if no Organization). Security Hub Account.2. | |
| CIS-AWS-7.0.0/2.2 | CIS | 1 | Keep account contact details current | Y | `aws_account_primary_contact` (`full_name`, `address_line_1`, `city`, `state_or_region`, `postal_code`, `country_code`, `phone_number`). Values come from variables, never from the repo. | `aws account get-contact-information` (manual review) | |
| CIS-AWS-7.0.0/2.3 | CIS | 1 | Register a security contact | Y | `aws_account_alternate_contact` (`alternate_contact_type = "SECURITY"`, `name`, `title`, `email_address`, `phone_number`) | `aws account get-alternate-contact --alternate-contact-type SECURITY`. Security Hub Account.1. | |
| CIS-AWS-7.0.0/2.4 | CIS | 1 | No access keys on the root user | Y, manual (EX-02) | none (Terraform cannot manage the root user) | `aws iam get-account-summary --query 'SummaryMap.AccountAccessKeysPresent'` equals 0. Security Hub IAM.4. | |
| CIS-AWS-7.0.0/2.5 | CIS | 1 | MFA on the root user | Y, manual (EX-02) | none | `aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'` equals 1. IAM.9. | |
| CIS-AWS-7.0.0/2.6 | CIS | 2 | Hardware MFA on the root user | Opt, manual (EX-02) | none | credential report row `<root_account>`; IAM.6 | 2.5 |
| CIS-AWS-7.0.0/2.7 | CIS | 1 | Do not use root for daily or administrative work | Y, manual | none | `aws iam get-credential-report` (decode, check `password_last_used` and `access_key_1_last_used_date` for the root row; do not print the report) | |
| CIS-AWS-7.0.0/2.8 | CIS | 1 | IAM password policy: minimum length 14 or more | Y | `aws_iam_account_password_policy` (`minimum_password_length = 14`, `allow_users_to_change_password = true`) | `aws iam get-account-password-policy`. IAM.15. | |
| CIS-AWS-7.0.0/2.9 | CIS | 1 | Password policy prevents reuse (24 per Prowler's check name) | Y | same resource, `password_reuse_prevention = 24` | same. IAM.16. | 2.8 |
| CIS-AWS-7.0.0/2.10 | CIS | 1 | MFA for every IAM user with a console password | Y (design: no IAM users) | none. Use short-lived role or SSO credentials as the spec says. | `aws iam list-users` returns none; else credential report. IAM.5. | |
| CIS-AWS-7.0.0/2.11 | CIS | 1 | Disable credentials unused for 45 days or more | Y | none | credential report columns `password_last_used`, `access_key_*_last_used_date`. IAM.22. | |
| CIS-AWS-7.0.0/2.12 | CIS | 1 | Rotate access keys within 90 days | Y (design: no long-lived keys) | none | `aws iam list-access-keys` per user. IAM.3. | |
| CIS-AWS-7.0.0/2.13 | CIS | 1 | Grant IAM users permissions through groups or roles only | Y | none. Do not use `aws_iam_user_policy` or `aws_iam_user_policy_attachment`. | `aws iam list-attached-user-policies --user-name <u>`. IAM.2. | |
| CIS-AWS-7.0.0/2.14 | CIS | 1 | No customer or AWS managed policy attached that allows `*:*` | Dev (EX-11) for the deployer; Y for everything the module creates | The module's roles use scoped policies (`aws_iam_policy_document`). No `AdministratorAccess` on module roles. | `aws iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess`. IAM.1 (Security Hub dropped it from v5.0.0; Prowler still checks it). | |
| CIS-AWS-7.0.0/2.15 | CIS | 1 | A support role exists to manage incidents with AWS Support | Y | `aws_iam_role` (trust policy for the account's own principals, `data.aws_caller_identity`), `aws_iam_role_policy_attachment` to `arn:aws:iam::aws:policy/AWSSupportAccess` | `aws iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AWSSupportAccess`. IAM.18. | |
| CIS-AWS-7.0.0/2.16 | CIS | 2 | EC2 instances use IAM instance roles | Y | Node role (`aws_iam_role`, trust `ec2.amazonaws.com`) used by `aws_eks_node_group.node_role_arn` | `aws ec2 describe-instances --query 'Reservations[].Instances[].IamInstanceProfile'` is non-null | |
| CIS-AWS-7.0.0/2.17 | CIS | 1 | Remove expired SSL/TLS certificates stored in IAM | Y | none. Do not upload certificates to IAM. | `aws iam list-server-certificates`. IAM.26. | |
| CIS-AWS-7.0.0/2.18 | CIS | 1 | IAM Access Analyzer (external access) enabled in all regions | Y for `us-west-2`; other regions Opt | `aws_accessanalyzer_analyzer` (`analyzer_name`, `type = "ACCOUNT"`). Do not use the `*_UNUSED_ACCESS` types (paid). | `aws accessanalyzer list-analyzers --region us-west-2`. IAM.28. | |
| CIS-AWS-7.0.0/2.19 | CIS | 2 | Centralize IAM user management through federation | Y, manual | none (credentials are short-lived per the spec) | `aws iam list-users`, `aws iam list-saml-providers` | |
| CIS-AWS-7.0.0/2.20 | CIS | 1 | `AWSCloudShellFullAccess` is not attached to any identity | Y | none | `aws iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AWSCloudShellFullAccess`. IAM.27. | |
| CIS-AWS-7.0.0/2.21 | CIS | 1 | Resource policies do not allow `Principal: "*"` without restriction | Y, manual | Every `aws_iam_policy_document`, `aws_s3_bucket_policy`, `aws_kms_key.policy` the module writes uses a named principal. The TLS-only S3 policy uses `*` only with `Effect = Deny`, which is allowed. | IAM Access Analyzer findings: `aws accessanalyzer list-findings-v2 --analyzer-arn <arn>` | 2.18 |

### 5.2 Logging and monitoring (CIS AWS, sections 4 and 5, and CIS EKS section 2)

Source: Prowler `cis_7.0_aws.json`, Security Hub CSPM controls (CloudTrail, Config, EC2.6, S3.22, S3.23, ELB.5, CloudWatch.1 to 14, SecurityHub), kube-bench `eks-1.8.0` `controlplane.yaml`.

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-AWS-7.0.0/4.1 | CIS | 1 | CloudTrail enabled in all regions, with read and write management events | Y | `aws_cloudtrail` (`is_multi_region_trail = true`, `include_global_service_events = true`, `s3_bucket_name`, `kms_key_id`, `enable_logging = true`) with `advanced_event_selector` for `eventCategory = Management`. Do not mix `event_selector` and `advanced_event_selector` on one trail. | `aws cloudtrail describe-trails`; `aws cloudtrail get-trail-status --name <t>`; `aws cloudtrail get-event-selectors --trail-name <t>`. CloudTrail.1. | trail bucket |
| CIS-AWS-7.0.0/4.2 | CIS | 2 | Log file validation on | Y | `enable_log_file_validation = true` | `describe-trails` field `LogFileValidationEnabled`. CloudTrail.4. | 4.1 |
| CIS-AWS-7.0.0/4.3 | CIS | 2 | AWS Config recording in all regions | Opt (cost; EX-16) | `aws_iam_service_linked_role` (`aws_service_name = "config.amazonaws.com"`); `aws_config_configuration_recorder` (`role_arn` = that role, `recording_group { all_supported = true, include_global_resource_types = true }`, optional `recording_mode`); `aws_config_delivery_channel` (`s3_bucket_name`); `aws_config_configuration_recorder_status` (`is_enabled = true`, `depends_on` the delivery channel) | `aws configservice describe-configuration-recorders`; `describe-configuration-recorder-status`; `describe-delivery-channels`. Config.1. | LOCAL/BKT-CONFIG |
| CIS-AWS-7.0.0/4.4 | CIS | 1 | Server access logging on the CloudTrail S3 bucket | Y | `aws_s3_bucket_logging` (`bucket` = trail bucket, `target_bucket` = LOCAL/BKT-LOG, `target_prefix`) | `aws s3api get-bucket-logging --bucket <b>`. CloudTrail.7. | LOCAL/BKT-LOG |
| CIS-AWS-7.0.0/4.5 | CIS | 2 | CloudTrail logs encrypted with a customer managed KMS key | Y | `kms_key_id` on `aws_cloudtrail` = LOCAL/KMS-LOGS | `describe-trails` field `KmsKeyId`. CloudTrail.2. | LOCAL/KMS-LOGS |
| CIS-AWS-7.0.0/4.6 | CIS | 2 | Rotation on every customer managed symmetric key | Y | `enable_key_rotation = true` on every `aws_kms_key` | `aws kms list-keys` then `describe-key` (`KeyManager = CUSTOMER`) then `get-key-rotation-status`. KMS.4. | |
| CIS-AWS-7.0.0/4.7 | CIS | 2 | VPC flow logging on in every VPC | Y (volume cost) | `aws_flow_log` (`vpc_id`, `traffic_type = "ALL"`, `log_destination_type = "s3"` and `log_destination` = flow-log bucket ARN, or `"cloud-watch-logs"` with `log_destination` and `iam_role_arn`; `max_aggregation_interval`) | `aws ec2 describe-flow-logs --filter Name=resource-id,Values=<vpc-id>`. EC2.6. | LOCAL/KMS-S3 |
| CIS-AWS-7.0.0/4.8 | CIS | 2 | Object-level write event logging for S3 | Opt (volume; EX-14) | `advanced_event_selector` on the trail: `field_selector` `eventCategory` equals `Data`; `resources.type` equals `AWS::S3::Object`; `readOnly` equals `false` | `get-event-selectors`. S3.22. | 4.1 |
| CIS-AWS-7.0.0/4.9 | CIS | 2 | Object-level read event logging for S3 | Opt (volume; EX-14) | same selector with `readOnly` equals `true`, or one selector without `readOnly` for both | `get-event-selectors`. S3.23. | 4.1 |
| CIS-AWS-7.0.0/4.10 | CIS | 1 | Access logging on AWS-managed web front ends (load balancers) | Y, with a caveat (EX-04) | NLB attribute `access_logs.s3.enabled=true`, `access_logs.s3.bucket`, `access_logs.s3.prefix`, set through the Service annotation `service.beta.kubernetes.io/aws-load-balancer-attributes` (needs AWS Load Balancer Controller). Bucket policy for the regional ELB log delivery principal. UNVERIFIED: whether NLB access logs record TCP-passthrough listeners. I recall they cover only TLS listeners but did not confirm it; read the NLB access log page before relying on this. | `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <arn>`. ELB.5. | LOCAL/BKT-LB |
| CIS-AWS-7.0.0/5.1 | CIS | 2 | Metric filter and alarm: unauthorized API calls | Opt | see pattern table below. `aws_cloudwatch_log_metric_filter` (`log_group_name` = trail log group, `pattern`, `metric_transformation { name, namespace, value = "1" }`) and `aws_cloudwatch_metric_alarm` (`metric_name`, `namespace`, `statistic = "Sum"`, `threshold = 1`, `comparison_operator = "GreaterThanOrEqualToThreshold"`, `alarm_actions` = LOCAL/SNS-ALERT). | `aws logs describe-metric-filters --log-group-name <g>`; `aws cloudwatch describe-alarms`. CloudWatch.2 (v1.2.0 mapping). | 5.0 |
| CIS-AWS-7.0.0/5.2 | CIS | 1 | Console sign-in without MFA | Y | pattern table | same. CloudWatch.3. | 5.0 |
| CIS-AWS-7.0.0/5.3 | CIS | 1 | Root account usage | Y | pattern table | same. CloudWatch.1. | 5.0 |
| CIS-AWS-7.0.0/5.4 | CIS | 1 | IAM policy changes | Y | pattern table | same. CloudWatch.4. | 5.0 |
| CIS-AWS-7.0.0/5.5 | CIS | 1 | CloudTrail configuration changes | Y | pattern table | same. CloudWatch.5. | 5.0 |
| CIS-AWS-7.0.0/5.6 | CIS | 2 | Console authentication failures | Opt | pattern table | same. CloudWatch.6. | 5.0 |
| CIS-AWS-7.0.0/5.7 | CIS | 2 | Disabling or scheduled deletion of customer keys | Opt | pattern table | same. CloudWatch.7. Teardown will trigger it; see EX-05. | 5.0 |
| CIS-AWS-7.0.0/5.8 | CIS | 1 | S3 bucket policy changes | Y | pattern table | same. CloudWatch.8. | 5.0 |
| CIS-AWS-7.0.0/5.9 | CIS | 2 | AWS Config configuration changes | Opt | pattern table | same. CloudWatch.9. | 5.0, 4.3 |
| CIS-AWS-7.0.0/5.10 | CIS | 2 | Security group changes | Opt | pattern table | same. CloudWatch.10. | 5.0 |
| CIS-AWS-7.0.0/5.11 | CIS | 2 | Network ACL changes | Opt | pattern table | same. CloudWatch.11. | 5.0 |
| CIS-AWS-7.0.0/5.12 | CIS | 1 | Network gateway changes | Y | pattern table | same. CloudWatch.12. | 5.0 |
| CIS-AWS-7.0.0/5.13 | CIS | 1 | Route table changes | Y | pattern table | same. CloudWatch.13. | 5.0 |
| CIS-AWS-7.0.0/5.14 | CIS | 1 | VPC changes | Y | pattern table | same. CloudWatch.14. | 5.0 |
| CIS-AWS-7.0.0/5.15 | CIS | 1 | AWS Organizations changes | N/A (EX-17) unless in an Organization | pattern: UNVERIFIED (not on the Security Hub pages read; take it from the CIS audit text) | same. No Security Hub control. | 5.0 |
| CIS-AWS-7.0.0/5.16 | CIS | 2 | AWS Security Hub enabled | Opt | `aws_securityhub_account` (`enable_default_standards = false`, `control_finding_generator = "SECURITY_CONTROL"`); `aws_securityhub_standards_subscription` (`standards_arn = "arn:aws:securityhub:us-west-2::standards/cis-aws-foundations-benchmark/v/5.0.0"`; the provider's own table lists only up to 3.0.0, so the 5.0.0 ARN is taken from the Security Hub page and is untested with this provider), `depends_on` the account | `aws securityhub describe-hub`; `aws securityhub get-enabled-standards` | 4.3 |
| CIS-EKS-1.8.0/2.1.1 | CIS | ? | Enable EKS control plane audit logs | Y | `aws_eks_cluster.enabled_cluster_log_types = ["api","audit","authenticator","controllerManager","scheduler"]` (minimum `["audit"]` for Security Hub EKS.8; the owner may trim for cost) | `aws eks describe-cluster --name <c> --query 'cluster.logging'`. EKS.8. | |
| CIS-EKS-1.8.0/2.1.2 | CIS | ? | Collect and manage the audit logs | Y | `aws_cloudwatch_log_group` named `/aws/eks/<cluster>/cluster` created before the cluster (`retention_in_days`, `kms_key_id` = LOCAL/KMS-LOGS), `aws_eks_cluster` with `depends_on` it. | `aws logs describe-log-groups --log-group-name-prefix /aws/eks/` shows retention and `kmsKeyId`. CloudWatch.16. | 2.1.1 |
| AWS-EKS-BP/DETECT-1 | AWS | | Threat detection on the cluster | Opt | `aws_guardduty_detector` (`enable = true`, `finding_publishing_frequency`); `aws_guardduty_detector_feature` with `name = "EKS_AUDIT_LOGS"`, `status = "ENABLED"`; optional `name = "EKS_RUNTIME_MONITORING"` (with `additional_configuration` for the managed agent add-on) and `S3_DATA_EVENTS`, `EBS_MALWARE_PROTECTION`. Use the feature resources; the detector's `datasources` block is deprecated. | `aws guardduty list-detectors`; `aws guardduty get-detector --detector-id <id>`. GuardDuty.1, .5, .7, .10 (these are Security Hub FSBP controls, not CIS). | |
| LOCAL/LOGGROUP-RETENTION | LOCAL | | Every log group the module creates has retention and KMS | Y | `aws_cloudwatch_log_group` (`retention_in_days`, `kms_key_id`) | `aws logs describe-log-groups` and check `retentionInDays` and `kmsKeyId` on all | |

#### Pattern table for rows 5.1 to 5.14

The exact filter patterns are published on the Security Hub CSPM CloudWatch controls page (`https://docs.aws.amazon.com/securityhub/latest/userguide/cloudwatch-controls.html`), one per control, and match the CIS audit text. This document does not copy them. The module should hold them in one `locals` map keyed by control number, so the verification tool can read the map. Dependencies for all rows: the trail must send to CloudWatch Logs (`aws_cloudtrail.cloud_watch_logs_group_arn` and `cloud_watch_logs_role_arn`), with an IAM role that trusts `cloudtrail.amazonaws.com` and allows `logs:CreateLogStream` and `logs:PutLogEvents` on that log group. Security Hub removed `CloudTrail.5` from the 5.0.0 standard but the metric filters cannot work without it.

### 5.3 Networking and VPC (CIS AWS section 6, CIS EKS 5.4)

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-AWS-7.0.0/6.1.1 | CIS | 1 | EBS encryption by default in each region | Y | `aws_ebs_encryption_by_default` (`enabled = true`); `aws_ebs_default_kms_key` (`key_arn` = LOCAL/KMS-EBS) | `aws ec2 get-ebs-encryption-by-default --region us-west-2`; `aws ec2 get-ebs-default-kms-key-id`. EC2.7. | LOCAL/KMS-EBS |
| CIS-AWS-7.0.0/6.1.2 | CIS | 1 | CIFS (port 445) not open to the internet | Y | no security group rule opens 445 | `aws ec2 describe-security-groups --filters Name=ip-permission.from-port,Values=445 Name=ip-permission.cidr,Values=0.0.0.0/0` returns none | |
| CIS-AWS-7.0.0/6.2 | CIS | 1 | No network ACL allows ingress from 0.0.0.0/0 to admin ports (22, 3389) | Y | `aws_default_network_acl` (`default_network_acl_id` = `aws_vpc.main.default_network_acl_id`) with explicit rules, or `aws_network_acl` plus `aws_network_acl_rule` for the subnets. The AWS default ACL allows all ingress, so the default must be replaced, not left. | `aws ec2 describe-network-acls --filters Name=vpc-id,Values=<vpc-id>`. EC2.21. | |
| CIS-AWS-7.0.0/6.3 | CIS | 1 | No security group allows ingress from 0.0.0.0/0 to admin ports | Y | Use `aws_vpc_security_group_ingress_rule`. No rule for 22 or 3389. NLB-facing rules open only 80, 443, 4318. | `aws ec2 describe-security-groups --filters Name=ip-permission.cidr,Values=0.0.0.0/0` then check ports. EC2.53. | |
| CIS-AWS-7.0.0/6.4 | CIS | 1 | Same for `::/0` | Y | same; the VPC is IPv4 only unless the owner chooses otherwise | `... Name=ip-permission.ipv6-cidr,Values=::/0`. EC2.54. | |
| CIS-AWS-7.0.0/6.5 | CIS | 2 | The default security group of every VPC allows no traffic | Y | `aws_default_security_group` (`vpc_id`) with no `ingress` and no `egress` blocks | `aws ec2 describe-security-groups --filters Name=group-name,Values=default`. EC2.2. | |
| CIS-AWS-7.0.0/6.6 | CIS | 2 | Peering route tables follow least access | N/A | no peering | | |
| CIS-AWS-7.0.0/6.7 | CIS | 1 | Instance metadata service allows only IMDSv2 | Y | launch template `metadata_options { http_endpoint = "enabled", http_tokens = "required", http_put_response_hop_limit = 1 }`. Optionally `aws_ec2_instance_metadata_defaults` (`http_tokens = "required"`, `http_put_response_hop_limit = 1`) for the whole region. | `aws ec2 describe-instances --query 'Reservations[].Instances[].MetadataOptions'`. EC2.8, EC2.170. | node LT |
| CIS-AWS-7.0.0/6.8 | CIS | 2 | VPC endpoints for AWS service access | Opt (EX-15). S3 gateway endpoint is Y. | `aws_vpc_endpoint` (`vpc_id`, `service_name = "com.amazonaws.us-west-2.s3"`, `vpc_endpoint_type = "Gateway"`, `route_table_ids`); interface endpoints with `vpc_endpoint_type = "Interface"`, `private_dns_enabled = true`, `subnet_ids`, `security_group_ids`, `policy` | `aws ec2 describe-vpc-endpoints` | |
| CIS-EKS-1.8.0/5.4.3 | CIS | ? | Nodes are private | Y | node group `subnet_ids` = private subnets; subnets with `map_public_ip_on_launch = false`; route `0.0.0.0/0` to `aws_nat_gateway`; public subnets only for the NLB. Tags `kubernetes.io/role/internal-elb = 1` on private and `kubernetes.io/role/elb = 1` on public subnets, `kubernetes.io/cluster/<name> = shared`. | `aws ec2 describe-subnets --query 'Subnets[].[SubnetId,MapPublicIpOnLaunch]'`; `aws ec2 describe-instances` shows no `PublicIpAddress`. EC2.15, EC2.9. | |
| LOCAL/VPC-DNS | LOCAL | | EKS needs DNS support and hostnames | Y | `aws_vpc` (`enable_dns_support = true`, `enable_dns_hostnames = true`) | `aws ec2 describe-vpc-attribute --attribute enableDnsHostnames --vpc-id <id>` | |
| LOCAL/VPC-NAT | LOCAL | | One NAT gateway (cost) | Y | `aws_nat_gateway`, `aws_eip`. One AZ only. A single NAT is a deliberate availability trade (EX-12). | `aws ec2 describe-nat-gateways` | |

### 5.4 EKS control plane (CIS EKS sections 2, 4.1.7 and 5)

Source: kube-bench `eks-1.8.0` (`managedservices.yaml`, `policies.yaml`, `controlplane.yaml`), Security Hub EKS controls, AWS EKS best-practices guide. Levels are `?` because the PDF was not readable. kube-bench marks all of section 5 as manual and unscored, so `kube-bench` cannot verify them; use the AWS CLI.

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-EKS-1.8.0/5.3.1 | CIS | ? | Kubernetes secrets encrypted with a customer managed KMS key | Y | `aws_eks_cluster.encryption_config { resources = ["secrets"], provider { key_arn = LOCAL/KMS-EKS } }`. Set at creation; adding it later is supported but takes time. | `aws eks describe-cluster --name <c> --query 'cluster.encryptionConfig'` | LOCAL/KMS-EKS |
| CIS-EKS-1.8.0/5.4.1 | CIS | ? | Restrict access to the control plane endpoint | Y | `vpc_config { endpoint_public_access = true, public_access_cidrs = [var.operator_cidr], endpoint_private_access = true }`. `public_access_cidrs` has no default and must not contain `0.0.0.0/0`. | `aws eks describe-cluster --name <c> --query 'cluster.resourcesVpcConfig'`. EKS.1 will fail while the public endpoint is on (EX-01). | |
| CIS-EKS-1.8.0/5.4.2 | CIS | ? | Private endpoint on, public off | Dev (EX-01) | `endpoint_public_access = false` when the operator reaches the VPC by tunnel or VPN | same | 5.4.1 |
| CIS-EKS-1.8.0/5.4.3 | CIS | ? | Private nodes | Y | see section 5.3 row of the same ID | | |
| CIS-EKS-1.8.0/5.4.4 | CIS | ? | Network policy enabled | Y | `aws_eks_addon` for `vpc-cni` with `configuration_values = jsonencode({ enableNetworkPolicy = "true" })` (the key is in the VPC CNI network policy page). Default-deny `NetworkPolicy` objects per namespace are in-cluster work (see 4.3.2). | `aws eks describe-addon --cluster-name <c> --addon-name vpc-cni --query 'addon.configurationValues'`; `kubectl get networkpolicy -A` | |
| CIS-EKS-1.8.0/5.4.5 | CIS | ? | Encrypt traffic to load balancers with TLS | Dev (EX-04) | NLB listeners are TCP. 443 passes through to ingress-nginx, which serves the certificate. 4318 passes through to Podium's mutual TLS. 80 is plain HTTP for the HTTP-01 challenge and the redirect. | `aws elbv2 describe-listeners --load-balancer-arn <arn>` | |
| CIS-EKS-1.8.0/5.5.1 | CIS | ? | Authenticate with the AWS IAM authenticator (or AWS CLI at or above the stated version) | Y | `aws_eks_cluster.access_config { authentication_mode = "API" }`; kubeconfig from `aws eks update-kubeconfig` uses `aws eks get-token` | `aws eks describe-cluster --name <c> --query 'cluster.accessConfig'` | |
| CIS-EKS-1.8.0/4.1.7 | CIS | ? | Use the Cluster Access Manager API (access entries) | Y | `aws_eks_access_entry` (`cluster_name`, `principal_arn`, `type = "STANDARD"`) and `aws_eks_access_policy_association` (`policy_arn` to an EKS access policy, `access_scope`). Then `bootstrap_cluster_creator_admin_permissions = false` so no hidden admin remains. Create the operator's entry first or the cluster becomes unreachable (EX-26). | `aws eks list-access-entries --cluster-name <c>` | |
| CIS-EKS-1.8.0/5.2.1 | CIS | ? | Dedicated service accounts with IAM roles, not the node role | Y | `aws_eks_addon` for `eks-pod-identity-agent`; `aws_eks_pod_identity_association` (`cluster_name`, `namespace`, `service_account`, `role_arn`) for `ebs-csi-controller-sa` in `kube-system`; role trust principal `pods.eks.amazonaws.com` with `sts:AssumeRole` and `sts:TagSession`. The same for the VPC CNI (`aws-node`): UNVERIFIED that the CNI add-on takes a Pod Identity association on 1.37; the EKS add-on page would confirm. | `aws eks list-pod-identity-associations --cluster-name <c>` | |
| CIS-EKS-1.8.0/5.1.1 to 5.1.4 | CIS | ? | ECR scanning, ECR access limits, read-only ECR for the cluster, approved registries only | N/A for 5.1.1 to 5.1.3 (no ECR; EX-18). 5.1.4 manual. | none for ECR. For 5.1.4 enforce approved registries in-cluster with a `ValidatingAdmissionPolicy` or a policy engine (in-cluster, not Terraform). Scan images in the build script instead of ECR. | `kubectl get pods -A -o jsonpath='{..image}'` and compare with an approved list | |
| STIG-K8S-V2R6/CNTR-K8-001162 | STIG | | Secrets encrypted at rest | Y | same as CIS-EKS 5.3.1 | same | |
| AWS-EKS-BP/VERSION-1 | AWS | | Cluster on a supported (standard support) Kubernetes version | Y | `aws_eks_cluster.version = "1.37"`, `upgrade_policy { support_type = "STANDARD" }` | `aws eks describe-cluster --name <c> --query 'cluster.[version,upgradePolicy]'`. EKS.2. | |
| AWS-EKS-BP/ADDONS-1 | AWS | | Cluster add-ons managed as EKS add-ons so they are versioned | Y | `aws_eks_addon` (`addon_name`, `addon_version`, `resolve_conflicts_on_update = "OVERWRITE"`) for `vpc-cni`, `coredns`, `kube-proxy`, `aws-ebs-csi-driver`, `eks-pod-identity-agent` | `aws eks list-addons --cluster-name <c>` | |
| AWS-EKS-BP/TAGS-1 | AWS | | Cluster and node group tagged | Y | `tags` / provider `default_tags` | `aws eks describe-cluster --query 'cluster.tags'`. EKS.6. | |
| LOCAL/EKS-DELPROT | LOCAL | | Cluster deletion protection | Dev (EX-05) | `aws_eks_cluster.deletion_protection = false` by default, variable-controlled | `aws eks describe-cluster --query 'cluster.deletionProtection'` | |
| LOCAL/EKS-ROLE | LOCAL | | Cluster IAM role | Y | `aws_iam_role` (trust `eks.amazonaws.com`), `aws_iam_role_policy_attachment` to `AmazonEKSClusterPolicy`; `depends_on` it from the cluster | `aws iam list-attached-role-policies --role-name <r>` | |

### 5.5 EKS in-cluster controls (CIS EKS sections 4 and 3.x policies; GitOps, not Terraform)

These are Kubernetes objects. Argo CD should deliver them. Verification uses `kubectl` with read access to non-secret resources, or `kube-bench` targets `policies` and `controlplane`.

| ID | Src | Lvl | Requirement | Needed | Implementation (in-cluster) | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-EKS-1.8.0/4.1.1 | CIS | ? | `cluster-admin` only where required | Dev (EX-07) | Argo CD controller uses a broad role. Bind no human or workload account to `cluster-admin` otherwise. | `kubectl get clusterrolebindings -o json` then list subjects of `cluster-admin`. kube-bench target `policies`. | |
| CIS-EKS-1.8.0/4.1.2 | CIS | ? | Minimize access to secrets | Y | Roles grant `get`/`list`/`watch` on `secrets` only to cert-manager and Argo CD in their namespaces. The Kubernetes MCP role denies Secrets (spec P4.4). | `kubectl auth can-i list secrets --as=<subject> -A` | |
| CIS-EKS-1.8.0/4.1.3 | CIS | ? | Minimize wildcards in Roles and ClusterRoles | Dev (EX-07) | Argo CD's role uses wildcards. All custom roles list explicit resources. | kube-bench `policies` 4.1.3 | |
| CIS-EKS-1.8.0/4.1.4 | CIS | ? | Minimize access to create pods | Y | Only controllers that deploy workloads hold `create pods`. | kube-bench 4.1.4 | |
| CIS-EKS-1.8.0/4.1.5 | CIS | ? | Default service accounts not used | Y | Each workload names its own ServiceAccount. `automountServiceAccountToken: false` on `default` in each namespace. | kube-bench 4.1.5 | |
| CIS-EKS-1.8.0/4.1.6 | CIS | ? | Service account tokens mounted only where necessary | Y | `automountServiceAccountToken: false` in pod specs that do not call the API (Podium) | kube-bench 4.1.6 | |
| CIS-EKS-1.8.0/4.1.8 to 4.1.12 | CIS | ? | Limit bind, impersonate, escalate; PersistentVolume creation; node proxy; webhook configuration objects; service account token creation | Y, manual | Review each role for these verbs. cert-manager's webhook configuration is expected. | `kubectl get clusterroles -o json` then search the verbs | |
| CIS-EKS-1.8.0/4.2.1 | CIS | ? | Minimize admission of privileged containers | Dev (EX-06) | Pod Security Admission labels on namespaces (`pod-security.kubernetes.io/enforce`): `restricted` for Podium, cert-manager and Argo CD; `baseline` or `restricted` for ingress-nginx; `privileged` only for `kube-system` (aws-node, kube-proxy, EBS CSI node plugin). UNVERIFIED that every chart runs under `restricted`; test each. | kube-bench 4.2.1; `kubectl get pods -A -o json` and look for `securityContext.privileged` | |
| CIS-EKS-1.8.0/4.2.2 | CIS | ? | No host PID namespace | Y | PSA `baseline` or stricter | kube-bench 4.2.2 | |
| CIS-EKS-1.8.0/4.2.3 | CIS | ? | No host IPC namespace | Y | PSA `baseline` or stricter | kube-bench 4.2.3 | |
| CIS-EKS-1.8.0/4.2.4 | CIS | ? | No host network namespace | Dev (EX-06) | `aws-node` and `kube-proxy` use it in `kube-system`. Nothing else. | kube-bench 4.2.4 | |
| CIS-EKS-1.8.0/4.2.5 | CIS | ? | `allowPrivilegeEscalation` not allowed | Y | PSA `restricted` forbids it | kube-bench 4.2.5 | |
| CIS-EKS-1.8.0/4.3.1 | CIS | ? | The CNI supports network policy | Y | VPC CNI with `enableNetworkPolicy` (see 5.4.4) | `kubectl get pods -n kube-system -l k8s-app=aws-node` and the add-on config | 5.4.4 |
| CIS-EKS-1.8.0/4.3.2 | CIS | ? | Every namespace has a network policy | Y | A default-deny `NetworkPolicy` plus explicit allows in `argocd`, `cert-manager`, `ingress-nginx`, `podium`, `kube-system` where supported. cert-manager HTTP-01 solver pods need ingress from the controller. | kube-bench 4.3.2; `kubectl get networkpolicy -A` | 4.3.1 |
| CIS-EKS-1.8.0/4.4.1 | CIS | ? | Prefer secrets as files over environment variables | Dev (EX-08) | Podium's token is read from an environment variable by Podium. Mount other secrets as files. | kube-bench 4.4.1 | |
| CIS-EKS-1.8.0/4.4.2 | CIS | ? | Consider external secret storage | Dev (EX-09) | not used | | |
| CIS-EKS-1.8.0/4.5.1 | CIS | ? | Namespaces create administrative boundaries | Y | One namespace per component | `kubectl get ns` | |
| CIS-EKS-1.8.0/4.5.2 | CIS | ? | The `default` namespace is not used | Y | no workload in `default` | kube-bench 4.5.2; `kubectl get all -n default` | |

### 5.6 EKS node group, launch template and kubelet (CIS EKS section 3, CIS AWS 6.7, AL2023)

Node OS hardening (Amazon Linux 2023) is the Ansible role's job and is outside this document except for the AWS-facing parts below. The node-level kubelet checks run with kube-bench on the node.

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| LOCAL/NG-1 | LOCAL | | Managed node group on the Amazon Linux 2023 EKS-optimized AMI, amd64, one node | Y | `aws_eks_node_group` (`ami_type = "AL2023_x86_64_STANDARD"`, `instance_types = [var.node_instance_type]`, `capacity_type = "ON_DEMAND"`, `scaling_config { desired_size = 1, min_size = 1, max_size = 1 }`, `update_config { max_unavailable = 1 }`, `launch_template { id, version }`, `subnet_ids` = private subnets, `node_role_arn`). Do not set `remote_access` (it conflicts with `launch_template` and would open SSH) or `disk_size`. | `aws eks describe-nodegroup --cluster-name <c> --nodegroup-name <n> --query 'nodegroup.[amiType,instanceTypes,scalingConfig]'` | |
| CIS-AWS-7.0.0/6.1.1 (node volumes) | CIS | 1 | Encrypted root volume with the customer key | Y | `aws_launch_template.block_device_mappings { device_name = "/dev/xvda", ebs { encrypted = true, kms_key_id = LOCAL/KMS-EBS, volume_type = "gp3", delete_on_termination = true, volume_size } }`. UNVERIFIED: that `/dev/xvda` is the root device name for the AL2023 EKS AMI; check `aws ec2 describe-images` for `RootDeviceName`. | `aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=<i> --query 'Volumes[].[Encrypted,KmsKeyId]'`. EC2.3, EC2.181. | LOCAL/KMS-EBS |
| CIS-AWS-7.0.0/6.7 (node) | CIS | 1 | IMDSv2 required, hop limit 1 so pods cannot reach node credentials | Y | `metadata_options` as in section 5.3. The AWS EKS best-practices guide recommends hop limit 1; it also warns that pods which need IMDS (some add-ons) will then fail, which is why Pod Identity is used. | `describe-launch-template-versions --launch-template-id <id> --versions '$Latest' --query 'LaunchTemplateVersions[0].LaunchTemplateData.MetadataOptions'` | CIS-EKS 5.2.1 |
| LOCAL/NG-NOPUBIP | LOCAL | | Launch template does not assign public IPs | Y | no `network_interfaces { associate_public_ip_address = true }` in the template (omit the block) | EC2.25 | |
| LOCAL/NG-SSM | LOCAL | | Node is managed by Systems Manager and tagged for the hardening association | Y | node role policy attachment `arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore`; launch template `tag_specifications` for `instance` and `volume` with the hardening tag | `aws ssm describe-instance-information`. SSM.1. | LOCAL/TF-TAGS |
| LOCAL/NG-ROLE | LOCAL | | Node role has only worker, SSM and image pull rights | Y | attachments: `AmazonEKSWorkerNodePolicy`, `AmazonSSMManagedInstanceCore`. No `AmazonEKS_CNI_Policy` on the node role when the CNI uses Pod Identity. Registry pull policy only if ECR is used; the project's images come from a public registry, so none is needed. | `aws iam list-attached-role-policies --role-name <r>` | |
| CIS-EKS-1.8.0/3.1.1 to 3.1.4 | CIS | ? | Kubelet kubeconfig and config file permissions 644 or stricter and owner `root:root` | Y | EKS AL2023 AMI defaults. If a check fails, fix in the Ansible role. | `kube-bench run --benchmark eks-1.8.0 --targets node --json` | |
| CIS-EKS-1.8.0/3.2.1 | CIS | ? | Kubelet anonymous auth disabled | Y | AMI default. If not, `NodeConfig.spec.kubelet.config.authentication.anonymous.enabled = false` in launch template `user_data` (nodeadm `NodeConfig`, `apiVersion node.eks.aws/v1alpha1`; the nodeadm API page says `spec.kubelet.config` is merged with defaults). `user_data` is a MIME multipart document for AL2023; UNVERIFIED exact wrapper, check the nodeadm documentation. | kube-bench 3.2.1 | |
| CIS-EKS-1.8.0/3.2.2 | CIS | ? | Kubelet authorization mode is not AlwaysAllow (Webhook) | Y | AMI default; override with `authorization.mode = "Webhook"` | kube-bench 3.2.2 | |
| CIS-EKS-1.8.0/3.2.3 | CIS | ? | Client CA file configured | Y | AMI default | kube-bench 3.2.3 | |
| CIS-EKS-1.8.0/3.2.4 | CIS | ? | Kubelet read-only port disabled | Y | `readOnlyPort = 0` (AMI default; override if not) | kube-bench 3.2.4 | |
| CIS-EKS-1.8.0/3.2.5 | CIS | ? | Streaming connection idle timeout is not 0 | Y | `streamingConnectionIdleTimeout = "4h0m0s"` | kube-bench 3.2.5 | |
| CIS-EKS-1.8.0/3.2.6 | CIS | ? | Kubelet manages iptables chains | Y | `makeIPTablesUtilChains = true` | kube-bench 3.2.6 | |
| CIS-EKS-1.8.0/3.2.7 | CIS | ? | Event record QPS 0 or a level that captures events | Y | `eventRecordQPS` set to a non-restrictive value | kube-bench 3.2.7 | |
| CIS-EKS-1.8.0/3.2.8 | CIS | ? | Kubelet client certificate rotation on | Y | `rotateCertificates = true` | kube-bench 3.2.8 | |
| CIS-EKS-1.8.0/3.2.9 | CIS | ? | Kubelet serving certificate rotation on | Y | feature gate `RotateKubeletServerCertificate` and `serverTLSBootstrap = true` where the AMI does not do it. UNVERIFIED: whether `serverTLSBootstrap` needs CSR approval on EKS. | kube-bench 3.2.9 | |
| AWS-EKS-BP/NG-PATCH | AWS | | Node group on a current AMI and Kubernetes version | Y | pin `release_version` deliberately; update on a schedule. AL2023 EKS nodes are replaced, not patched in place. | `aws eks describe-nodegroup --query 'nodegroup.[version,releaseVersion]'`. EKS.9. | |
| LOCAL/SSM-ASSOC | LOCAL | | SSM association runs the Ansible playbook, output logged and private | Y | `aws_ssm_association` (`name`, `association_name`, `targets { key = "tag:<hardening-tag>", values = [...] }`, `schedule_expression`, `output_location { s3_bucket_name, s3_key_prefix }`); playbook object in LOCAL/BKT-PLAYBOOK | `aws ssm list-associations`; `aws ssm describe-association-execution-targets`. SSM.3. | |
| LOCAL/SSM-DOCS-PRIVATE | LOCAL | | SSM documents are not public; public sharing blocked | Y | `aws_ssm_service_setting` for the document public-sharing setting with value `Disable`. UNVERIFIED: the setting ID. Read the SSM.7 remediation text before use. | SSM.4, SSM.7 | |
| STIG-AL2023-V1R4/V-283441 | STIG | | Enable FIPS mode | Dev (see C-02) | not applied | | |
| STIG-AL2023-V1R4/V-274038 | STIG | | SSH installed (with the sshd rules V-274042 to V-274051) | Dev (see C-01) | no SSH; access is Session Manager | | |
| STIG-AL2023-V1R4/V-274027, V-274028 | STIG | | firewalld installed and active | Dev (see C-04) | not applied; security groups and network policy filter traffic | | |
| STIG-AL2023-V1R4/V-274022, V-274023, V-274175 | STIG | | chrony installed, enabled, syncing at least daily | Y | AL2023 ships chrony and the Amazon Time Sync Service; confirm in the Ansible role | `systemctl is-enabled chronyd` via SSM | |
| STIG-AL2023-V1R4/V-274017, V-274167, V-274067 | STIG | | auditd installed, audit processes start early, at least a week of local audit records | Y | Ansible role | `rpm -q audit` via SSM | |
| STIG-AL2023-V1R4/V-274147, V-274157, V-274155, V-274186 | STIG | | Account lockout after 3 failed logons, lock held until released, pam_faillock loaded | N/A (no interactive accounts; no SSH) | Ansible role may still set it | | |
| STIG-AL2023-V1R4/V-274137, V-274133, V-274138, V-274140, V-274161 | STIG | | Password length 15, complexity and dictionary rules | N/A (no password logons) | | | |
| STIG-AL2023-V1R4/V-274065, V-274066 | STIG | | DoD notice and consent banner | Dev (see C-03) | not applied | | |

### 5.7 Encryption and KMS (summary of rows that also appear elsewhere)

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-AWS-7.0.0/4.6 | CIS | 2 | Rotation on all customer keys | Y | see section 5.2 | | |
| LOCAL/KMS-POLICY | LOCAL | | Key policies grant no wildcard principal and no wildcard decrypt | Y | `aws_kms_key.policy` with named principals and `kms:ViaService` or `aws:SourceArn` conditions. IAM policies do not allow `kms:Decrypt` on `*`. | `aws kms get-key-policy --key-id <id> --policy-name default`. KMS.1, KMS.2, KMS.5 (Security Hub FSBP). | |
| LOCAL/KMS-DELETE | LOCAL | | Keys are not deleted unintentionally | Dev (EX-05) | `deletion_window_in_days = 7` (the minimum) so teardown works; the demo accepts the risk | KMS.3 | |
| LOCAL/EBS-PVC | LOCAL | | Persistent volumes are encrypted with the customer key | Y | In-cluster `StorageClass` `gp3` with `parameters: { type: gp3, encrypted: "true", kmsKeyId: <KMS-EBS ARN> }`. The EBS CSI role needs `kms:CreateGrant`, `kms:GenerateDataKeyWithoutPlaintext`, `kms:Decrypt`, `kms:DescribeKey` on LOCAL/KMS-EBS. | `aws ec2 describe-volumes --filters Name=tag:kubernetes.io/created-for/pvc/name,Values=<pvc>` check `Encrypted` | LOCAL/KMS-EBS |

### 5.8 S3, including the Terraform state bucket (CIS AWS section 3.1)

Applies to every bucket the module creates: state, access-log target, CloudTrail, Config, flow logs, Ansible playbook, load balancer logs.

| ID | Src | Lvl | Requirement | Needed | Terraform | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-AWS-7.0.0/3.1.1 | CIS | 2 | Bucket policy denies plain HTTP | Y | `aws_s3_bucket_policy` with a statement `Effect = "Deny"`, `Principal = "*"`, `Action = "s3:*"`, bucket and object ARNs, condition `Bool: aws:SecureTransport = "false"`. Merge with other statements in one `aws_iam_policy_document`; a bucket has one policy. | `aws s3api get-bucket-policy --bucket <b>`. S3.5. | |
| CIS-AWS-7.0.0/3.1.2 | CIS | 2 | MFA delete on versioned buckets | Dev (EX-03) | not applied. `aws_s3_bucket_versioning.versioning_configuration.mfa_delete` exists but needs the root user's MFA device and breaks lifecycle rules. | S3.20 | |
| CIS-AWS-7.0.0/3.1.3 | CIS | 2 | Discover and classify S3 data | Opt, manual (Macie) | `aws_macie2_account` (`status = "ENABLED"`) only if the owner wants it | `aws macie2 get-macie-session`. Macie.1. | |
| CIS-AWS-7.0.0/3.1.4 | CIS | 1 | S3 Block Public Access on, account level and bucket level | Y | `aws_s3_account_public_access_block` (all four arguments true); `aws_s3_bucket_public_access_block` per bucket (`block_public_acls`, `block_public_policy`, `ignore_public_acls`, `restrict_public_buckets` all true) | `aws s3control get-public-access-block --account-id <id>` (id taken from `aws sts get-caller-identity` inside the tool, never printed); `aws s3api get-public-access-block --bucket <b>`. S3.1, S3.8. | |
| CIS-AWS-7.0.0/3.2.1 to 3.2.4 | CIS | 1 | RDS encryption, auto minor upgrades, not public, Multi-AZ | N/A | no RDS | | |
| CIS-AWS-7.0.0/3.3.1 | CIS | 1 | EFS encryption | N/A | no EFS (EBS CSI only) | | |
| LOCAL/BKT-STATE | LOCAL | | State bucket hardening beyond the CIS rows | Y | `aws_s3_bucket_server_side_encryption_configuration` (`sse_algorithm = "aws:kms"`, `kms_master_key_id` = LOCAL/KMS-S3, `bucket_key_enabled = true`); `aws_s3_bucket_versioning` (`status = "Enabled"`); `aws_s3_bucket_ownership_controls` (`object_ownership = "BucketOwnerEnforced"`); `aws_s3_bucket_logging` to LOCAL/BKT-LOG; `aws_s3_bucket_lifecycle_configuration` expiring noncurrent versions after a set number of days and aborting incomplete uploads; policy statement denying principals other than the operator role and the deployer. Native locking (`use_lockfile = true`) writes a `.tflock` object, so the operator needs `s3:PutObject` and `s3:DeleteObject` on that key. `force_destroy = true` so teardown can remove versions (spec P1.3). The Podium token is in this state (spec section 4), so KMS encryption is required. | `get-bucket-encryption`, `get-bucket-versioning`, `get-bucket-logging`, `get-bucket-ownership-controls`, `get-bucket-lifecycle-configuration`. S3.9, S3.10, S3.12, S3.13, S3.14, S3.17. | LOCAL/KMS-S3, LOCAL/BKT-LOG |
| LOCAL/BKT-TRAIL | LOCAL | | CloudTrail bucket | Y | as LOCAL/BKT-LOG with SSE-KMS allowed, plus policy statements for `cloudtrail.amazonaws.com` (`s3:GetBucketAcl` on the bucket, `s3:PutObject` on the prefix with `s3:x-amz-acl = bucket-owner-full-control` and `aws:SourceArn`) | same | |
| LOCAL/BKT-CONFIG | LOCAL | | Config delivery bucket | Opt (with Config) | policy for `config.amazonaws.com` | same | |
| LOCAL/BKT-LB | LOCAL | | Load balancer log bucket | Opt | policy for the regional ELB log delivery principal; SSE-S3 | same | |
| LOCAL/BKT-PLAYBOOK | LOCAL | | Playbook bucket for the SSM association | Y | as the state bucket; `aws_s3_object` for the playbook; read access for the node role limited to that prefix | same | |
| LOCAL/BKT-FLOW | LOCAL | | Flow log bucket | Y | policy for `delivery.logs.amazonaws.com`, SSE-S3 or SSE-KMS with matching key policy | same | |

### 5.9 Load balancer (NLB created by the ingress Service)

The NLB is created by Kubernetes when the ingress-nginx Service of `type: LoadBalancer` is applied, not by Terraform. The listener design is in the spec (80, 443, 4318). Hardening is done through Service annotations and, optionally, a Terraform-managed security group. The Service annotation names below were checked against the AWS Load Balancer Controller annotations page (controller v3.6.1, released 2026-10-21). The in-tree cloud provider does not support all of them, which is why the controller is listed as optional in section 4 and required here for these rows.

| ID | Src | Lvl | Requirement | Needed | Implementation | Verify | Dep |
|---|---|---|---|---|---|---|---|
| LOCAL/NLB-SCHEME | LOCAL | | Internet-facing NLB with only ports 80, 443, 4318 | Y | annotations `service.beta.kubernetes.io/aws-load-balancer-type: external`, `aws-load-balancer-scheme: internet-facing`, `aws-load-balancer-nlb-target-type: ip` or `instance` | `aws elbv2 describe-listeners --load-balancer-arn <arn>` shows exactly those three ports | |
| LOCAL/NLB-SG | LOCAL | | NLB has a security group limiting inbound ports | Y | annotation `aws-load-balancer-security-groups`; `aws_security_group` plus `aws_vpc_security_group_ingress_rule` for 80, 443, 4318 only; annotation `aws-load-balancer-manage-backend-security-group-rules` | `aws elbv2 describe-load-balancers --query 'LoadBalancers[].SecurityGroups'` | CIS-AWS 6.3 |
| CIS-AWS-7.0.0/4.10 | CIS | 1 | Access logs | Y with caveat | annotation `aws-load-balancer-attributes: access_logs.s3.enabled=true,access_logs.s3.bucket=<b>,access_logs.s3.prefix=<p>` | see section 5.2 | LOCAL/BKT-LB |
| LOCAL/NLB-DELPROT | LOCAL | | Deletion protection | Dev (EX-05) | `deletion_protection.enabled=true` in the attributes annotation. With it on, the controller cannot delete the NLB during teardown. Default false. | ELB.6 | |
| CIS-EKS-1.8.0/5.4.5 | CIS | ? | TLS to load balancers | Dev (EX-04) | passthrough, see section 5.4 | | |
| LOCAL/INGRESS-TLS | LOCAL | | Ingress terminates TLS 1.2 and 1.3 only, hides version | Y | ingress-nginx ConfigMap keys `ssl-protocols: "TLSv1.2 TLSv1.3"`, `server-tokens: "false"`, `ssl-ciphers` set to a modern list. UNVERIFIED here: key names come from the ingress-nginx ConfigMap documentation, which was not read. The NGINX STIG V1R1 (32 findings) was not read either, so any further NGINX requirements are UNVERIFIED. | `kubectl get cm -n ingress-nginx -o yaml`; `openssl s_client -tls1_1` fails against the hostname | |
| LOCAL/INGRESS-ARCHIVED | LOCAL | | Controller is maintained | Dev (question 1) | The repository is archived and receives no security fixes. | `gh api repos/kubernetes/ingress-nginx --jq .archived` | |

### 5.10 Secrets

| ID | Src | Lvl | Requirement | Needed | Implementation | Verify | Dep |
|---|---|---|---|---|---|---|---|
| CIS-EKS-1.8.0/5.3.1 | CIS | ? | etcd secrets encrypted with a customer key | Y | see section 5.4 | | |
| STIG-K8S-V2R6/CNTR-K8-001160 | STIG | | Secrets not stored as environment variables | Dev (EX-08, C-06) | Podium reads `PODIUM_TOKEN` from the environment | | |
| LOCAL/SECRET-STATE | LOCAL | | Secret values that Terraform writes live only in encrypted state | Y | `random_password` (not in this document's provider check), `kubernetes_secret` with `data`; state bucket per LOCAL/BKT-STATE; mark outputs `sensitive = true` | `aws s3api get-bucket-encryption` on the state bucket | LOCAL/BKT-STATE |
| LOCAL/SECRET-ARGO | LOCAL | | Argo CD initial admin secret removed or admin disabled after setup | Y | in-cluster | `kubectl get secret argocd-initial-admin-secret -n argocd` must be absent (existence check only, never read the value) | |
| STIG-K8S-V2R6/CNTR-K8-001163 | STIG | | Secret access limited to need-to-know | Y | see CIS-EKS 4.1.2 | | |
| CIS-AWS-7.0.0/2.21 | CIS | 1 | No unrestricted principals on Secrets Manager, KMS, S3, SQS, SNS, Lambda, EventBridge policies | Y | no Secrets Manager in the build; other policies per section 5.1 | | |

### 5.11 Kubernetes STIG V2R6 dispositions (first 50 of 93 checks)

Checks 51 to 93 were not read. The 50 below came from a third-party audit file; the disposition column is this document's own judgment. Codes: **M** AWS manages the control plane component and the customer has no setting (EKS shared responsibility; evidence is the AWS compliance programs, UNVERIFIED which reports apply); **N** a node or kubelet item, covered by the CIS EKS row named; **K** an in-cluster item covered by the CIS EKS row named; **D** DoD-specific, not applicable.

| ID | Requirement (short) | Disp | Covered by |
|---|---|---|---|
| STIG-K8S-V2R6/CNTR-K8-000150 | Controller Manager uses TLS 1.2 or later | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000160 | Scheduler uses TLS 1.2 or later | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000170 | API server uses TLS 1.2 or later | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000180 | etcd uses TLS | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000190 | etcd uses TLS | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000220 | Unique service accounts per workload | K | CIS-EKS 4.1.5 |
| STIG-K8S-V2R6/CNTR-K8-000270 | API server uses Node and RBAC authorization | M | EKS (access entries, 5.5.1) |
| STIG-K8S-V2R6/CNTR-K8-000290 | User resources in dedicated namespaces | K | CIS-EKS 4.5.1 |
| STIG-K8S-V2R6/CNTR-K8-000300 | Scheduler secure binding | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000310 | Controller Manager secure binding | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000330 | Kubelet read-only port disabled | N | CIS-EKS 3.2.4 |
| STIG-K8S-V2R6/CNTR-K8-000350 | API server secure port set | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000360 | API server anonymous auth disabled | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000370 | Kubelet anonymous auth disabled | N | CIS-EKS 3.2.1 |
| STIG-K8S-V2R6/CNTR-K8-000380 | Kubelet explicit authorization | N | CIS-EKS 3.2.2 |
| STIG-K8S-V2R6/CNTR-K8-000400 | Worker nodes do not run sshd | N | no SSH (C-01); Ansible `service_facts` |
| STIG-K8S-V2R6/CNTR-K8-000410 | Worker nodes do not have sshd enabled | N | same |
| STIG-K8S-V2R6/CNTR-K8-000420 | Kubernetes dashboard not enabled | K | not installed; `kubectl get pods -A` shows none |
| STIG-K8S-V2R6/CNTR-K8-000430 | `kubectl cp` gives expected access | K | manual |
| STIG-K8S-V2R6/CNTR-K8-000440 | Kubelet `staticPodPath` does not enable static pods | N | Ansible check of `/etc/kubernetes/manifests` and kubelet config |
| STIG-K8S-V2R6/CNTR-K8-000450 | DynamicAuditing gate not enabled | N | UNVERIFIED whether this gate still exists in 1.37 |
| STIG-K8S-V2R6/CNTR-K8-000460 | DynamicKubeletConfig gate not enabled | N | same |
| STIG-K8S-V2R6/CNTR-K8-000470 | Alpha APIs disabled | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000610 | API server audit log path set | M | CIS-EKS 2.1.1 (EKS ships audit logs to CloudWatch) |
| STIG-K8S-V2R6/CNTR-K8-000700 | Audit record content | M | CIS-EKS 2.1.1 |
| STIG-K8S-V2R6/CNTR-K8-000850 | Kubelet denies hostname override | N | UNVERIFIED how EKS nodes set it; check kubelet flags |
| STIG-K8S-V2R6/CNTR-K8-000860 | Manifests owned by root | N | Ansible file check |
| STIG-K8S-V2R6/CNTR-K8-000880 | KubeletConfiguration owned by root | N | CIS-EKS 3.1.4 |
| STIG-K8S-V2R6/CNTR-K8-000890 | KubeletConfiguration permissions 644 or stricter | N | CIS-EKS 3.1.3 |
| STIG-K8S-V2R6/CNTR-K8-000900 | Manifest files least privilege | N | Ansible file check |
| STIG-K8S-V2R6/CNTR-K8-000910 | Controller Manager profiling disabled | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-000920 | API server enforces PPSM CAL ports and protocols | D | DoD port registry; not applicable |
| STIG-K8S-V2R6/CNTR-K8-000930 | Scheduler enforces PPSM CAL | D | same |
| STIG-K8S-V2R6/CNTR-K8-000940 | Controllers enforce PPSM CAL | D | same |
| STIG-K8S-V2R6/CNTR-K8-000950 | etcd enforces PPSM CAL | D | same |
| STIG-K8S-V2R6/CNTR-K8-000960 | User pods use non-privileged host ports | K | PSA `baseline` (CIS-EKS 4.2.x) |
| STIG-K8S-V2R6/CNTR-K8-001160 | Secrets not stored as environment variables | K | CIS-EKS 4.4.1 (EX-08) |
| STIG-K8S-V2R6/CNTR-K8-001161 | Sensitive data in Secrets or an external store | K | CIS-EKS 4.4.2 |
| STIG-K8S-V2R6/CNTR-K8-001162 | Secrets encrypted at rest | K | CIS-EKS 5.3.1 |
| STIG-K8S-V2R6/CNTR-K8-001163 | Secret access limited | K | CIS-EKS 4.1.2 |
| STIG-K8S-V2R6/CNTR-K8-001300 | Kubelet does not disable timeouts | N | CIS-EKS 3.2.5 |
| STIG-K8S-V2R6/CNTR-K8-001360 | User functionality separated | K | CIS-EKS 4.5.1 |
| STIG-K8S-V2R6/CNTR-K8-001400 | API server uses approved cipher suites | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-001410 | API server SSL CA set | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-001420 | Kubelet SSL CA set | N | CIS-EKS 3.2.3 |
| STIG-K8S-V2R6/CNTR-K8-001430 | Controller Manager SSL CA set | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-001440 | API server has a communication certificate | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-001450 | etcd client authentication enabled | M | EKS |
| STIG-K8S-V2R6/CNTR-K8-001460 | Kubelet `tlsPrivateKeyFile` set | N | CIS-EKS 3.2.9 (UNVERIFIED mapping) |
| STIG-K8S-V2R6/CNTR-K8-001470 | Kubelet `tlsCertFile` set | N | same |

## 6. Account-level threat detection and posture services: summary of what is not in the CIS AWS list

GuardDuty, Inspector and Macie are not required by the CIS AWS Foundations rows that were read (only Macie appears, as a manual L2 row). They are listed because the AWS EKS best-practices guide's "Detective Controls" and "Runtime Security" sections recommend threat detection, and the owner asked for threat detection to be considered. The page of that guide was read only as a section index (it now redirects to the AWS documentation), so the specific recommendations are UNVERIFIED. Rows `AWS-EKS-BP/DETECT-1` (section 5.2) and the section 4 inventory carry the detail.

## 7. Conflicts (CIS and STIG disagree, so CIS was chosen)

"Confidence" says how much of each side was read. For every row CIS was chosen.

| ID | Item | STIG says | CIS says | Confidence | Result |
|---|---|---|---|---|---|
| C-01 | SSH on nodes | AL2023 STIG V-274038 requires SSH to be installed, with rules V-274042 to V-274051 for its configuration. Kubernetes STIG CNTR-K8-000400 and CNTR-K8-000410 require that worker nodes do not run or enable sshd. The two STIGs disagree with each other. | CIS AWS 6.3 and 6.4 forbid admin ports open to the world. The CIS AL2023 text on SSH was not readable (UNVERIFIED), but no CIS source read requires SSH to exist. | STIG sides read. CIS-AL2023 side UNVERIFIED. | No SSH. Access is Session Manager. No `remote_access` on the node group. |
| C-02 | FIPS mode on nodes | AL2023 STIG V-283441 and V-283452 require FIPS mode and a FIPS system-wide crypto policy. | No CIS control read requires FIPS. CIS AL2023 side UNVERIFIED. | STIG read. CIS UNVERIFIED. | FIPS is not enabled. It is for regulated environments, and an EKS-optimized AMI would need a FIPS variant. |
| C-03 | Logon banner text | AL2023 STIG V-274065 and V-274066 require the DoD notice and consent banner. | CIS Linux benchmarks usually require a banner that does not disclose OS details. UNVERIFIED for CIS AL2023. | STIG read. CIS UNVERIFIED. | No DoD banner. Use the CIS wording once confirmed. |
| C-04 | Host firewall | AL2023 STIG V-274027 and V-274028 require `firewalld` installed and active. | CIS AL2023 firewall section not readable (UNVERIFIED). On Kubernetes, kube-proxy and the CNI program iptables; a second manager can break them. CIS EKS 3.2.6 requires the kubelet to manage iptables chains. | STIG read. CIS UNVERIFIED. | No `firewalld`. Security groups and network policy filter traffic (EX-10). |
| C-05 | Password length | AL2023 STIG V-274137 requires 15 characters for OS accounts. | CIS AWS 2.8 requires 14 for IAM users. | Both read. Different objects (OS accounts vs IAM users), so this is not a real conflict. | IAM policy uses 14. No OS password logons exist. |
| C-06 | Secrets in environment variables | Kubernetes STIG CNTR-K8-001160 states secrets must not be stored as environment variables. | CIS EKS 4.4.1 says to prefer files (title: "Prefer using secrets as files over secrets as environment variables"). It is a preference, not a ban. | Both titles read, not full text. | Podium's token is an environment variable. Allowed under CIS wording, recorded as EX-08. |
| C-07 | Cluster endpoint and DoD port registry | Kubernetes STIG CNTR-K8-000920 to 000950 require components to use only ports and protocols registered in a DoD list. | CIS EKS has no equivalent. | STIG read. | Not applicable outside DoD. |
| C-08 | Cloud mission-owner SRGs | Cloud Computing Mission Owner Network and OS SRGs set DoD cloud rules. | CIS has no equivalent. | Listed only; not read. | Not applicable. |

No conflict was found between a CIS AWS Foundations control and any STIG, because no STIG covers AWS services.

## 8. Exceptions and deviations

Each is a place where a control cannot be met as written, or where the project's design and the control point opposite ways. The verification tool records these as `not-applicable` with the `EX` ID in the evidence.

| ID | Controls | Why | What is done instead |
|---|---|---|---|
| EX-01 | CIS-EKS 5.4.2, Security Hub EKS.1 | The spec runs Terraform and `kubectl` from a workstation. A private-only API endpoint needs a VPN or tunnel into the VPC. | Public endpoint on, limited by `public_access_cidrs` to the operator, private endpoint also on for nodes. Revisit if a tunnel is added. |
| EX-02 | CIS-AWS 2.4, 2.5, 2.6 | The root user is outside Terraform. | Manual check with `aws iam get-account-summary`. The tool reports `manual` or reads the saved CLI output. |
| EX-03 | CIS-AWS 3.1.2, S3.20 | MFA delete needs the root user's MFA and cannot coexist with lifecycle rules. Teardown must empty versioned buckets. | Not applied. Versioning, SSE-KMS and a bucket policy protect the state. |
| EX-04 | CIS-EKS 5.4.5, Security Hub ELB.18, ELB.21, ELB.22 | The NLB listeners are TCP. TLS ends at ingress-nginx (443) and at Podium (4318, mutual TLS with its own CA). An NLB TLS listener would terminate TLS and break the client certificate on 4318. Port 80 must stay plain for HTTP-01. UNVERIFIED: whether NLB access logs record TCP listeners. | TCP passthrough with security group limits. |
| EX-05 | LOCAL/NLB-DELPROT, LOCAL/EKS-DELPROT, LOCAL/KMS-DELETE, CIS-AWS 5.7, Security Hub ELB.6, KMS.3 | The spec requires a tested teardown that returns the region to empty. Deletion protection and key deletion alarms get in the way. | Defaults off. The CMK deletion alarm will fire at teardown. |
| EX-06 | CIS-EKS 4.2.1, 4.2.4 | `aws-node`, `kube-proxy` and the EBS CSI node plugin need privileged containers or the host network. | Only `kube-system` is `privileged` under Pod Security Admission. |
| EX-07 | CIS-EKS 4.1.1, 4.1.3 | Argo CD's application controller needs wide rights to manage cluster resources. | Accept for the controller only. No other subject on `cluster-admin`. |
| EX-08 | CIS-EKS 4.4.1, STIG CNTR-K8-001160 | Podium takes its token from `PODIUM_TOKEN`. | Environment variable from a Secret reference, Secret encrypted in etcd by KMS. |
| EX-09 | CIS-EKS 4.4.2 | External secret storage (Secrets Manager with a CSI or operator) adds cost and components for one secret. | Not used. Accepted in the spec. |
| EX-10 | Node OS: IP forwarding, bridge filtering, host firewall, any CIS AL2023 control that blocks Kubernetes networking | Kubernetes needs `net.ipv4.ip_forward = 1` and related kernel settings. The CIS AL2023 numbers are UNVERIFIED (PDF not read). | The Ansible role skips them and lists each in its README and `docs/DECISIONS.md`, per spec section 4. |
| EX-11 | CIS-AWS 2.14, Security Hub IAM.1 | The identity that runs Terraform may carry `AdministratorAccess`. | Question 5. Produce a scoped deployer role where possible. The module itself attaches no `*:*` policy. |
| EX-12 | Single NAT gateway, single node, single AZ (Security Hub ELB.13) | Cost. Non-HA is a stated non-goal. | Accepted. Not a CIS control. |
| EX-14 | CIS-AWS 4.8, 4.9 | S3 data events cost per event. Terraform's state polling makes many. | Off unless the owner enables `enable_s3_data_events`. If off, those two L2 rows are non-compliant by choice. |
| EX-15 | CIS-AWS 6.8 | Interface endpoints cost per hour per AZ. | Gateway endpoint for S3 only. NAT carries the rest. |
| EX-16 | CIS-AWS 4.3, 5.9, 5.16 | Config and Security Hub cost per item and per check. | Owner decision (question 3). |
| EX-17 | CIS-AWS 2.1.x, 5.15, Account.2 | They concern AWS Organizations. | Not applicable if the account is standalone. |
| EX-18 | CIS-EKS 5.1.1 to 5.1.3 | The project does not use ECR. | Image scanning moves to the image build script. Pulls come from a public registry. |
| EX-20 | S3 server access logging target | The log bucket must use SSE-S3, not SSE-KMS, and does not log itself (Security Hub S3.9 expects suppression for it). | LOCAL/BKT-LOG. |
| EX-21 | Kubernetes 1.37 | CIS EKS 2.0.0 states support for 1.34 and 1.35. The 1.37 behavior is untested by the benchmark. | Run kube-bench as a guide, not proof. |
| EX-22 | STIG-K8S control-plane items marked M | AWS runs the control plane. No customer setting exists. | Record as `not-applicable` with reason "AWS-managed". |
| EX-24 | ingress-nginx archived | No security fixes. | Question 1. |
| EX-25 | Tooling | kube-bench has no `eks-2.0.0` config. | Use `eks-1.8.0` and flag the version gap in the output. |
| EX-26 | CIS-EKS 4.1.7 and `bootstrap_cluster_creator_admin_permissions = false` | Removing the creator's implicit admin rights can lock the operator out if the access entry is wrong. | Create the explicit entry in the same apply and test `kubectl auth whoami` before the next step. |

EX-13 and EX-19 and EX-23 are not used.

## 9. Verification and validation tool

### 9.1 Purpose

A command that compares the hardening list with evidence from an Ansible run, saved AWS CLI output, kube-bench output and a few `kubectl` queries, and prints one line per control. It never changes anything. It does not report a score. This is a project rule.

### 9.2 Control-list format

One YAML file, `hardening/controls.yaml` (path is a suggestion), generated from section 5 of this document by the implementer and then edited by hand. Every row of section 5 becomes one entry. Entries are the contract between the module and the tool.

```yaml
# JSON-schema-like description, in YAML
controls:                      # list
  - id: string                 # required, unique, e.g. CIS-AWS-7.0.0/6.7. Regex: ^(CIS|STIG|AWS|LOCAL)-[A-Za-z0-9.]+(-[A-Za-z0-9.]+)?/[A-Za-z0-9.\-]+$
    source:
      kind: CIS|STIG|AWS|LOCAL # required
      benchmark: string        # e.g. AWS Foundations, Amazon EKS
      version: string          # exact version used for the number
      control: string          # the benchmark's own number or name
      level: 1|2|null          # CIS Level, null if unknown or not CIS
    area: account-iam|logging|network|eks-control-plane|eks-nodes|encryption|s3|load-balancer|secrets|in-cluster
    title: string              # your own words, short
    applies: required|optional|not-applicable|deviation
    exception: string|null     # EX-nn when applies is deviation or not-applicable for a design reason
    reason: string|null        # required when applies is not-applicable or deviation
    cost_flag: none|fixed|hourly|volume
    depends_on: [id]           # other control ids; if a dependency fails, report this one as fail with evidence "dependency <id> failed"
    terraform:                 # for the module author; the tool does not read it
      - resource: string
        args: {key: value}
    checks:                    # evidence rules; empty list means manual
      - type: aws_cli|prowler|securityhub|kube_bench|ansible|kubectl|file
        input: string          # key of the evidence file, see 9.3
        selector: string       # JMESPath (aws_cli, securityhub, kubectl), check ID (prowler), test number (kube_bench), task-name prefix (ansible)
        expect:                # one of
          equals: any
          in: [any]
          matches: regex
          exists: bool
          not_empty: bool
          status: PASS         # for prowler/kube_bench/securityhub
        scope: all|any         # for lists: every item must match, or at least one
    combine: all|any           # how several checks combine; default all
    manual:                    # used when checks is empty or when applies is optional and disabled
      question: string
      attestation_key: string|null
```

Three example entries:

```yaml
- id: CIS-AWS-7.0.0/6.7
  source: {kind: CIS, benchmark: AWS Foundations, version: 7.0.0, control: "6.7", level: 1}
  area: eks-nodes
  title: Instances allow only IMDSv2
  applies: required
  cost_flag: none
  depends_on: []
  terraform:
    - resource: aws_launch_template
      args: {metadata_options: {http_endpoint: enabled, http_tokens: required, http_put_response_hop_limit: 1}}
  checks:
    - type: aws_cli
      input: ec2-describe-instances-nodes
      selector: "Reservations[].Instances[].MetadataOptions.HttpTokens"
      expect: {equals: required}
      scope: all
    - type: prowler
      input: prowler-ocsf
      selector: ec2_instance_imdsv2_enabled
      expect: {status: PASS}
  combine: all

- id: CIS-EKS-1.8.0/3.2.4
  source: {kind: CIS, benchmark: Amazon EKS, version: 1.8.0, control: "3.2.4", level: null}
  area: eks-nodes
  title: Kubelet read-only port is disabled
  applies: required
  cost_flag: none
  depends_on: []
  checks:
    - type: kube_bench
      input: kube-bench-node
      selector: "3.2.4"
      expect: {status: PASS}

- id: STIG-K8S-V2R6/CNTR-K8-000400
  source: {kind: STIG, benchmark: Kubernetes, version: V2R6, control: CNTR-K8-000400, level: null}
  area: eks-nodes
  title: Worker nodes do not run sshd
  applies: required
  cost_flag: none
  depends_on: []
  checks:
    - type: ansible
      input: ansible-json
      selector: "[STIG-K8S-V2R6/CNTR-K8-000400]"
      expect: {status: ok}
```

### 9.3 Inputs

All inputs are files in an evidence directory, so the tool itself needs no cloud credentials. A collector script (`scripts/collect-evidence.sh`, read-only commands, uses the named profile) writes them. The tool refuses to run if it detects a credentials file in the evidence directory.

| Input key | Produced by | Notes |
|---|---|---|
| `ansible-json` | `ANSIBLE_STDOUT_CALLBACK=json` (the `json` stdout callback; in current Ansible it ships in the `ansible.posix` collection, UNVERIFIED) on the SSM-run or local playbook, plus a second run in `--check` mode | Convention for the role: every task that enforces a control starts its `name` with `[<control ID>]`. A task that is `ok` in the second run is a pass. `changed` in check mode is drift and a fail. `failed` is a fail. `skipped` with a recorded reason is `not-applicable`. The role also writes the applied-ID file the spec requires (P3.1); the tool can cross-check the two. |
| `aws-*` files | `scripts/collect-evidence.sh` runs the commands in section 5, one JSON file each, `--output json` | Named by `input` key, for example `ec2-describe-instances-nodes.json`. |
| `prowler-ocsf` | `prowler aws --compliance cis_7.0_aws --output-formats json-ocsf` | Reuses Prowler's check IDs (they are in `cis_7.0_aws.json`). |
| `kube-bench-node`, `-policies`, `-controlplane`, `-managedservices` | `kube-bench run --benchmark eks-1.8.0 --targets <t> --json` run as a Job or via SSM on the node | Node target must run on the node. |
| `kubectl-*` | read-only `kubectl get ... -o json` for namespaces, pods, roles, bindings, network policies, service accounts. Never `get secret` (the spec denies it). | Existence checks for secrets use `kubectl get secret <name> -o name`. |
| `securityhub-findings` | optional, `aws securityhub get-findings` with a filter on the CIS standard | Cross-check only. Security Hub supports CIS 5.0.0, not 7.x. |
| `terraform-plan-json` | optional, produced by a `make` target and stripped of sensitive values | Pre-apply static checks of arguments against `terraform:` entries. `terraform show` is denied to Claude by the spec, so a person or `make` runs it. |
| `attestations.yaml` | a person | Signed-off answers for manual controls, with date and initials. |

### 9.4 Mapping results to control IDs

1. For each control, evaluate each `checks[]` entry against its input file with the selector and `expect`.
2. A check yields `pass`, `fail`, or `no-evidence` (file or selector missing). `no-evidence` is reported as `fail` with evidence "no evidence supplied". Silence never passes.
3. Combine with `combine` (default `all`).
4. If `applies` is `not-applicable` or `deviation`, status is `not-applicable` and the evidence line starts `deviation:<EX-nn>` or `n/a:<reason>`. The checks are still run and shown in `--verbose` so drift against an accepted deviation is visible.
5. If `checks` is empty, status is `manual`. If `attestations.yaml` holds a current answer for `attestation_key`, the evidence shows it, and the status stays `manual`.
6. If a dependency failed, the dependent control is `fail` with the dependency ID.
7. Optional controls disabled in the module (`enable_*` false) are `not-applicable` with `deviation:EX-14` (or the matching EX).

### 9.5 Output

One line per control ID, tab separated, sorted by ID: `ID<TAB>STATUS<TAB>evidence`. `STATUS` is exactly `pass`, `fail`, `not-applicable` or `manual`. The evidence is one line, at most 200 characters, naming the input and the observed value. Example:

```
CIS-AWS-7.0.0/6.7	pass	ec2-describe-instances-nodes: HttpTokens=required (1 instance)
CIS-EKS-1.8.0/3.2.4	fail	kube-bench-node: 3.2.4 FAIL readOnlyPort=10255
CIS-AWS-7.0.0/2.5	manual	root MFA is checked by a person; attestation dated 2026-10-30
```

There is no percentage, no pass rate and no weighted score. A final line may state the count for each status as integers (`fail=2 pass=61 not-applicable=19 manual=7`), never a ratio. `--format json` gives the same fields as an array. The header line names the control-list file hash and the benchmark versions, so a result is tied to what it was checked against.

### 9.6 Exit codes

| Code | Meaning |
|---|---|
| 0 | No control is `fail`. `manual` lines do not fail the run. |
| 1 | At least one control is `fail`. |
| 2 | Bad usage, schema error in the control list, unreadable evidence directory. |
| 3 | Internal error, or the redaction step could not run. Nothing is printed. |
| 4 | With `--strict`, no `fail` but at least one `manual` has no current attestation. |

### 9.7 What it must never print

- Credentials of any kind: access keys (`AKIA`, `ASIA` prefixes and their secrets), session tokens, kubeconfig contents, bearer tokens, JWTs (`eyJ` prefix), the Podium token, Argo CD passwords.
- AWS account IDs: any 12-digit number that follows `arn:aws:...::` or appears in `Account`, `OwnerId`, `UserId` fields. ARNs print as `arn:aws:iam::<acct>:role/name`.
- Secret values of any Kubernetes Secret. The tool never reads one.
- Raw evidence files. Only the selected value is printed, after redaction.

Redaction runs over every output line, including `--verbose`, error messages and stack traces. If the redactor fails, the tool exits 3 and prints nothing else. The tool should have a test that feeds known fake account IDs, keys and tokens through every output path and asserts none survive. The tool must also never log to a file anything it would not print.

### 9.8 Language

Python 3. Ansible already needs Python on the workstation, so no new runtime. PyYAML and `jsonschema` cover the control list, JSON handling is in the standard library, and JMESPath (the library the AWS CLI uses) gives the same selectors as the `--query` examples above. Prowler is also Python, so its OCSF output and check IDs can be read without glue. The repo's own scripts are Bash and `make`; the tool is one file called from a `make verify-hardening` target. Go would give a single binary but adds a toolchain this project does not otherwise use. Bash with `jq` would handle 20 controls but not the schema, redaction tests or dependency rules.

### 9.9 Existing tools checked

| Tool | What was checked | Version and date seen | Covers | Decision |
|---|---|---|---|---|
| Prowler | Repo file list, `cis_6.0_aws.json` and `cis_7.0_aws.json` read, latest release | 5.45.0, 2026-10-15 | Runs AWS checks mapped to CIS AWS 1.4 to 7.0 (files `cis_1.4` to `cis_7.0`). JSON/OCSF output. 7.1.0 not seen. | Wrap. Run it, read its output, reuse its check IDs. Do not re-implement. |
| kube-bench | Repo file list, `eks-1.8.0` node, policies, managedservices, controlplane YAML read, latest release | v0.16.0, 2026-08-05 | CIS EKS 1.8.0 node checks (automated), policies checks (automated), section 5 and 2 (manual only). Also `eks-stig-kubernetes-v1r6`. No 2.0.0 config. | Wrap. It is the only automated source for 3.x and 4.x. |
| Security Hub CSPM | Documentation read | CIS 5.0.0, 3.0.0, 1.4.0, 1.2.0 | Managed AWS checks, per-region, needs Config. | Use as a cross-check, not the source of truth. Version lags CIS. |
| Trivy | Release only | v0.76.1, 2026-10-21 | UNVERIFIED which compliance specs ship for EKS or Kubernetes | Not assessed. |
| Kubescape | Release only | v4.0.16, 2026-10-14 | UNVERIFIED which frameworks it ships for EKS | Not assessed. |
| Powerpipe / Steampipe `aws_compliance` mod | Search results only; the hub pages rendered nothing | not seen | Lists AWS CIS benchmarks, version coverage UNVERIFIED | Not assessed. |
| Ansible Lockdown AMAZON2023-CIS role | README read | based on CIS AL2023 1.0.0 (2023-06-26) | Applies CIS AL2023 controls with a control-ID-named variable per rule | Candidate for the node role. Check its license and that its rules skip the Kubernetes-breaking ones (EX-10). Not tested. |

Not checked: CIS-CAT Pro (needs CIS membership), OpenSCAP content, Checkov and other Terraform static analyzers.

Recommendation: write a thin Python tool that owns the control list, the mapping and the output, and wraps Prowler and kube-bench rather than replacing them. Everything those two do not cover (Terraform-specific settings, AWS-side items outside the CIS list, Ansible results, deviation tracking, redaction) is the tool's own work.

## 10. Acceptance checklist

Module author:

- [ ] Every row in section 5 with Needed `Y` has Terraform or an in-cluster manifest, or a documented reason it is manual.
- [ ] Every `Opt` row has an `enable_*` variable; defaults agree with the owner's answers to section 3.
- [ ] Every `Dev` row has an `EX` entry and appears in `docs/DECISIONS.md`.
- [ ] Account-level resources are in their own root with their own state (question 8).
- [ ] Every KMS key rotates; every log group has retention and a key; every bucket has Block Public Access, TLS-only policy, encryption, ownership controls.
- [ ] State bucket is encrypted with KMS, versioned, logged, lifecycle-managed, not public, and `use_lockfile` works against it.
- [ ] `terraform validate` and `terraform plan` pass with a `terraform.tfvars.example` that holds no account ID, ARN, email or address.
- [ ] A full apply and a full teardown were run in a throwaway account with approval; teardown left nothing behind (spec P6.2), including KMS keys scheduled for deletion, log groups, Config recorder, GuardDuty detector, Security Hub subscription.
- [ ] Control-list entries exist for every row; each ID matches the pattern in 9.2.

Verification tool:

- [ ] Prints one line per control and no ratio.
- [ ] Exit codes follow 9.6.
- [ ] Redaction test passes for account IDs, keys, JWTs, the Podium token.
- [ ] Missing evidence produces `fail`, not `pass`.
- [ ] A control marked as an accepted deviation prints the `EX` ID.
- [ ] kube-bench version gap (EKS 1.8.0 vs 2.0.0) is printed in the header.
- [ ] `--check`-mode Ansible drift is reported as `fail`.

Owner sign-off:

- [ ] Answers to questions 1 to 8 recorded.
- [ ] Someone with a CIS login confirmed the AWS Foundations 7.1.0 and EKS 2.0.0 numbering against sections 5.1 to 5.6.
- [ ] Root account controls (2.4, 2.5, 2.6) checked by a person.

## 11. Sources

All fetched or searched on 2026-10-24 unless stated. "Used" says what the source backed.

| Source | URL | Version or date | Used for |
|---|---|---|---|
| CIS Benchmarks index | https://www.cisecurity.org/cis-benchmarks | no dates shown; AWS Foundations 7.1.0, EKS 2.0.0, EKS Auto Mode 1.0.0, AL2023 1.0.0, Kubernetes 2.1.0, Docker 1.9.0 | current version numbers |
| CIS Amazon Web Services page | https://www.cisecurity.org/benchmark/amazon_web_services | AWS Foundations 7.1.0; Compute 2.0.0; Database 2.0.0; End User Compute 1.2.0; Storage 1.0.0 | current versions of the AWS benchmarks |
| CIS Amazon Linux page | https://www.cisecurity.org/benchmark/amazon_linux | AL2023 1.0.0 | node OS benchmark version |
| CIS Benchmarks June 2026 update | https://www.cisecurity.org/insights/blog/cis-benchmarks-june-2026-update | EKS 2.0.0, Kubernetes 2.0.1 for Kubernetes 1.34 and 1.35 | EKS version and range |
| CIS Benchmarks October 2026 update | https://www.cisecurity.org/insights/blog/cis-benchmarks-october-2026-update | | checked for AWS entries; none |
| CIS Benchmarks December 2025 update | https://www.cisecurity.org/insights/blog/cis-benchmarks-december-2025-update | EKS 1.8.0 | prior EKS version (search snippet) |
| Prowler compliance JSON, CIS AWS 7.0.0 | https://raw.githubusercontent.com/prowler-cloud/prowler/master/prowler/compliance/aws/cis_7.0_aws.json | v7.0.0 | control numbers, titles, Levels, automated or manual, check IDs for section 5.1 to 5.3 |
| Prowler compliance JSON, CIS AWS 6.0.0 | https://raw.githubusercontent.com/prowler-cloud/prowler/master/prowler/compliance/aws/cis_6.0_aws.json | v6.0.0 (first 100,000 characters) | confirmed numbering changed between 6 and 7 |
| Prowler repository and releases | https://github.com/prowler-cloud/prowler | 5.45.0, 2026-10-15 | tool review |
| Security Hub CSPM: CIS AWS Foundations | https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html | supports 5.0.0, 3.0.0, 1.4.0, 1.2.0 | control mapping, supported versions, ARNs |
| Security Hub CSPM controls pages: EKS, ELB, S3, EC2, IAM, CloudTrail, CloudWatch, KMS, GuardDuty, Config, SSM, Inspector, ECR, Secrets Manager, Account, Macie, Auto Scaling | https://docs.aws.amazon.com/securityhub/latest/userguide/<service>-controls.html | current docs | control IDs, titles, Config rule names, metric filter pages |
| kube-bench repository, configs | https://github.com/aquasecurity/kube-bench (`cfg/eks-1.8.0/*.yaml`, `cfg/eks-stig-kubernetes-v1r6/node.yaml`, `cfg/config.yaml`) | v0.16.0, 2026-08-05 | CIS EKS control numbers, titles, scored flags, audit approach |
| kube-bench platform table | https://github.com/aquasecurity/kube-bench/blob/main/docs/platforms.md | older rows only (up to eks-1.5.0 and STIG V1R6) | confirmed STIG config exists; table is out of date |
| STIG listing mirror | https://www.stigviewer.com/stigs and https://www.stigviewer.com/stigs/amazon_linux_2023 | AL2023 V1R4 2026-05-22; Kubernetes V2R6 2026-02-12; Container Platform SRG V2R4 2025-09-10; NGINX V1R1 2026-01-07; Cloud mission owner SRGs | which STIGs exist; AL2023 rule IDs |
| Kubernetes STIG V2R6 audit file (third-party scanner vendor) | https://www.tenable.com/audits/DISA_STIG_Kubernetes_v2r6 | v2r6, 6/9/2026; page 1 of 2 (50 of 93 checks) | rule IDs and titles for section 5.11 |
| DISA STIG downloads | https://public.cyber.mil/stigs/downloads/ (redirects to https://www.cyber.mil/stigs/downloads/) | | could not read; empty landing page |
| Terraform AWS provider documentation (markdown in repository) | https://github.com/hashicorp/terraform-provider-aws/tree/main/website/docs/r | release v6.70.0, 2026-10-22 | every resource and argument name in section 5 (`eks_cluster`, `eks_node_group`, `launch_template`, `cloudtrail`, `flow_log`, `s3_*`, `iam_account_password_policy`, `ebs_encryption_by_default`, `ebs_default_kms_key`, `kms_key`, `securityhub_*`, `guardduty_*`, `config_*`, `accessanalyzer_analyzer`, `cloudwatch_*`, `lb*`, `default_security_group`, `eks_addon`, `eks_access_entry`, `eks_pod_identity_association`, `ssm_association`, `macie2_account`, `budgets_budget` and others) |
| AWS Load Balancer Controller service annotations | https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/service/annotations.md | controller v3.6.1, 2026-10-21 | annotation names for NLB hardening |
| AWS EKS best-practices guide | https://aws.github.io/aws-eks-best-practices/security/docs/ and https://docs.aws.amazon.com/eks/latest/best-practices/identity-and-access-management.html | current | section index; IMDSv2 and hop limit guidance |
| AWS EKS user guide: cluster endpoint, VPC CNI network policy | https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html and .../cni-network-policy-configure.html | current | public access CIDRs; `enableNetworkPolicy` key |
| AWS S3 user guide: server access logging | https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html | current | SSE-S3 requirement on the log target bucket |
| nodeadm API reference | https://github.com/awslabs/amazon-eks-ami/blob/main/nodeadm/doc/api.md | current | `NodeConfig`, `spec.kubelet.config` merged with defaults |
| Kubernetes blog: Ingress NGINX retirement | https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/ | 2025-11-11 | retirement date. Repository state from the GitHub API: archived, last push 2026-03-23, last release `controller-v1.15.1` 2026-03-19 |
| CIS AWS Storage Services Benchmark v1.0.0 (mirrored PDF) | https://itsecure.hu/wp-content/uploads/2025/05/CIS_AWS_Storage_Services_Benchmark_v1.0.0.pdf | v1.0.0, header 07-03-2024 | table of contents only: all recommendations are manual |
| Ansible Lockdown AL2023 CIS role | https://github.com/ansible-lockdown/AMAZON2023-CIS | README says benchmark v1.0.0, 26-06-2023 | AL2023 benchmark date; candidate role |
| Trivy, Kubescape, Terraform AWS provider, LB controller releases | GitHub releases API for each repository | Trivy v0.76.1; Kubescape v4.0.16; provider v6.70.0; controller v3.6.1 | version numbers only |
| `docs/SPEC.md`, `PROJECT.md` | this repository | v0.5 of the spec | scope and design |

Not read: CIS PDFs for AWS Foundations (7.1.0, 7.0.0), EKS (2.0.0, 1.8.0), AL2023; DISA's own library; the full AL2023 STIG; page 2 of the Kubernetes STIG audit file; the NGINX STIG; the Container Platform SRG; AWS pricing pages; the ingress-nginx ConfigMap reference; the NLB access log page; the CloudTrail and CloudWatch Logs KMS pages.
