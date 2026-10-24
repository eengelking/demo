# SPEC: EKS platform bootstrap for Podium

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-24 | First draft. |
| 0.2 | 2026-10-24 | Cloudflare DNS moves out of Terraform into an agent and a skill. Adds the reporter port (4318) and the DNS-only hostname, agent-safe Terraform execution, the image build task, the Kubernetes MCP task, a single node on Kubernetes 1.37. |
| 0.3 | 2026-10-24 | cert-manager uses the HTTP-01 solver, so no Cloudflare token is stored in the cluster. |
| 0.4 | 2026-10-24 | Kubernetes 1.37 confirmed. Node is a `c6i.xlarge`. Kubernetes MCP server chosen. AWS credentials are the user's named profile. |
| 0.5 | 2026-10-24 | Kubernetes MCP server settings follow its current docs. Pinned to v0.0.69. |
| 0.6 | 2026-10-24 | Hardening becomes a first-class requirement: `docs/HARDENING-AWS.md` and `docs/HARDENING-LINUX.md` are normative inputs. Adds the account baseline and deployer role, Traefik in place of ingress-nginx, one load balancer for ports 80, 443 and 4318, SSH over a Session Manager tunnel with an Ansible-created key pair, the hardening validation tool, and a validated teardown of everything. Tasks renumbered into seven phases. |

This project stands up a hardened Kubernetes platform on AWS from nothing and deploys [Podium](https://github.com/eengelking/podium) onto it. Terraform creates the account baseline, the network and an EKS cluster. Ansible hardens the node and keeps it hardened as it is replaced. Argo CD is bootstrapped and deploys Podium from a Helm chart in this repo. cert-manager issues a Let's Encrypt certificate for `demo.edengelking.com`.

"Hardened" means the CIS benchmarks first, with the DISA STIGs used only where CIS is silent. This is not a government environment, so CIS wins any disagreement. The requirements are written in two guidelines, which this spec treats as normative:

- [`docs/HARDENING-AWS.md`](HARDENING-AWS.md): the AWS account, network, EKS, IAM, logging and encryption controls, the deployer role, the validation tool design and the teardown validation design.
- [`docs/HARDENING-LINUX.md`](HARDENING-LINUX.md): the Amazon Linux 2023 node controls (CIS Level 1 Server only), the upstream Ansible role to use with its overrides, the deviation register and the node-side evidence.

Where a task below says "per the guideline", the control rows in that document are the acceptance criteria, and a control is either implemented or recorded as a deviation with its reason.

It also shows that agents and skills can do specific work inside guardrails. A DNS agent (the what and why) and a DNS skill (the how) own the Cloudflare record through an MCP server, a Kubernetes MCP server lets an agent check cert-manager before and after a certificate is requested, and Claude Code runs Terraform without being able to read a secret.

One person runs it from a workstation with short-lived AWS credentials. The repo is public, and its history is meant to be read, so every stage is a separate, named, testable unit.

## 1. Goals and non-goals

### Goals

- One documented sequence takes an empty AWS account to Podium answering over HTTPS on `https://demo.edengelking.com`, with the reporter port reachable on `demo.edengelking.com:4318`.
- Every stage can be run, verified and torn down on its own.
- The account, the network, the cluster and the node meet the controls in the two hardening guidelines, and every exception is recorded as a deviation with a reason.
- A validation tool reads the hardening results and reports pass, fail, not applicable or manual per control ID, with no percentage score.
- Nodes are hardened after they join, and a replaced node is re-hardened without anyone running anything.
- The Let's Encrypt environment (`staging` or `prod`) is one variable, and the public hostname is one variable.
- The DNS record is created, changed and removed only by the DNS agent, using the DNS skill, and only for that one hostname.
- Claude Code can plan Terraform and run the approved targets without any credential, state value or token reaching its context.
- Teardown removes everything the tool created, including the account baseline, and a validation script proves it.
- No secret is ever in the repo, in Argo CD's UI or in a log.

### Non-goals (do NOT build these)

- High availability for Podium. It runs one replica on a SQLite volume.
- Multi-region, multi-cluster, multi-account or AWS Organizations support. The controls that need an Organization are recorded as not applicable.
- Managing Cloudflare with Terraform. The record belongs to the DNS agent.
- Cloudflare's proxy. The hostname is DNS-only (grey cloud), because the reporter port needs mutual TLS to reach the pod untouched.
- CIS Level 2 controls on the node. They are excluded and listed in the Linux guideline with the reason for each.
- A pre-baked hardened node image, Bottlerocket, or any node OS other than Amazon Linux 2023.
- A compliance score or percentage.
- Auto-approving Terraform against a real account. Every apply and destroy is a command a person approves.
- Reporting real Claude Code sessions to the deployed Podium. Only the Podium simulator reports to it.
- Authentication in front of the Podium console. Reads are open by design on a trusted network.
- Macie. It is off.
- Persisting anything after teardown. The account baseline is destroyed too.
- A hardware MFA key for the root account. The root controls are manual checks the owner does by hand.

## 2. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Infrastructure | Terraform 1.17 | State, plan review, dependency ordering and a reliable destroy. Teardown of everything is a hard requirement. |
| State | S3 bucket with native locking (`use_lockfile`) | No DynamoDB table to create and tear down. |
| Region and arch | `us-west-2`, amd64 | Chosen by the maintainer. |
| Kubernetes | EKS 1.37, one managed node | Chosen by the maintainer. |
| Node size | `c6i.xlarge` (4 vCPU, 8 GiB) | About $0.17 an hour on demand in `us-west-2` per a third-party listing, to be confirmed against AWS pricing. |
| Node OS | Amazon Linux 2023 EKS-optimized AMI | Current for EKS. A CIS benchmark and a maintained Ansible role exist. |
| Hardening | The upstream CIS Amazon Linux 2023 Ansible role (`ansible-lockdown/AMAZON2023-CIS`, pinned to a tag), Level 1 Server, with the overrides in `docs/HARDENING-LINUX.md`, applied through an SSM association | Re-applies to replaced nodes. A maintained role is better than a new one. |
| SSH | An EC2 key pair created by Ansible (`amazon.aws.ec2_key`), reached through a Session Manager tunnel | No inbound port 22. The private key never enters Terraform state. |
| Account baseline | A separate Terraform root with CloudTrail, Config, Security Hub, GuardDuty, Inspector, KMS, the password policy, the S3 account block and EBS default encryption | Required by the AWS guideline. Destroyed by teardown. |
| Deployer identity | A scoped IAM role the Makefile profile assumes, and a separate `operator-ssh` role for starting SSH sessions | The current identity is broader than the CIS controls allow, and the SSH permission should not widen the deployer. |
| Storage | EBS CSI driver add-on, `gp3` StorageClass | Podium's SQLite volume needs a PersistentVolumeClaim. |
| GitOps | Argo CD, app-of-apps, tracking `development` until the first release | `main` has nothing to sync before a release. |
| Ingress | Traefik v3, a current chart pinned, behind one NLB | Replaces ingress-nginx, which is archived. Terminates TLS on 443, and also carries a TCP entrypoint for 4318. |
| Reporter port | Traefik TCP entrypoint on 4318 with an `IngressRouteTCP` using `tls.passthrough` | Keeps one NLB and one hostname while Podium's mutual TLS sees the client certificate. Client-certificate behavior is documented by inference only, so it is tested first (P5.2). |
| Certificates | cert-manager, HTTP-01 solver, one `staging`/`prod` variable | No Cloudflare token in the cluster. |
| DNS | Cloudflare, DNS-only record, managed by an agent and a skill through the Cloudflare MCP server | Shows an agent with a narrow job and guardrails. |
| Cluster introspection | `containers/kubernetes-mcp-server` v0.0.69, pinned, run with a committed TOML config | The most-starred Kubernetes MCP server found. Its limits come from its config and from a role. |
| Hardening checks | A Python tool that wraps Prowler (AWS) and kube-bench (EKS) and reads Ansible output | Reuses maintained checks and owns the control list, the mapping and the output. |
| App packaging | A small Helm chart in this repo | Argo CD renders it. |
| Image registry | GitHub Container Registry, public package | No pull secret needed. |
| Scripting | Bash, `make` | One script or target per stage. |

## 3. Glossary

- **Stage:** one independently runnable unit of the build.
- **Root:** a Terraform root module with its own state.
- **Control:** one numbered requirement from a benchmark, STIG or local rule in the hardening guidelines.
- **Deviation:** a control that is not implemented as written, with the reason, recorded in the guideline and in `docs/DECISIONS.md`.
- **Control list:** the machine-readable YAML form of the guideline's controls, read by the validation tool.
- **Issuer variable:** `letsencrypt_env`, either `staging` or `prod`.
- **Hostname variable:** `podium_hostname`, set to `demo.edengelking.com`.
- **Reporter port:** port 4318, Podium's optional TLS listener that accepts only reporters with a client certificate.
- **DNS agent / DNS skill:** the agent that decides what the record should be and why, and the skill that carries out the change.

## 4. Architecture

### Layout

```
terraform/
  state/        S3 bucket for remote state (local state, created once)
  account/      deployer and operator roles, KMS, logging, detection, account-level settings, budget
  network/      VPC, subnets, NAT, endpoints, flow logs
  cluster/      EKS, node group, launch template, IAM, EBS CSI add-on, SSM association
  addons/       Argo CD, Traefik, cert-manager, issuers, Podium secret
ansible/
  roles/        the overrides wrapped around the upstream role
  playbooks/    the node hardening playbook and the key pair playbook
gitops/
  apps/         Argo CD Application manifests (app-of-apps)
  charts/podium/ Helm chart for Podium
hardening/
  controls/     machine-readable control lists (AWS, Linux)
  tool/         the validation tool and its tests
.claude/
  agents/       the DNS agent
  skills/       the DNS skill
scripts/        one script per stage, plus verify, teardown, token and checks
```

`network`, `cluster` and `addons` read each other's outputs through remote state. A stage never reads another stage's files.

### Traffic and access

```
browser   -- 443  --> NLB --> Traefik (Let's Encrypt cert) --> Podium :4317 (plain HTTP)
reporter  -- 4318 --> NLB --> Traefik TCP entrypoint (TLS passthrough) --> Podium :4318 (mutual TLS)
operator  -- SSH over Session Manager (no inbound port 22) --> node
operator  -- EKS public endpoint limited to the operator's /32 (and the private endpoint)
```

All traffic to `demo.edengelking.com` goes to one NLB through a DNS-only record. Podium's own CA and serving certificate live on its `/data` volume, and the serving certificate must include the hostname, so `PODIUM_ALLOWED_HOSTS` is set before Podium's first start.

### Account baseline and deployer role

The `account` root creates everything the AWS guideline requires at account level: the deployer role, the `operator-ssh` role, KMS keys, CloudTrail with data events, Config, Security Hub with the CIS standard, GuardDuty, Inspector, the IAM password policy, the S3 account public access block and EBS default encryption. These are destroyed with everything else. The deployer role is created once by an admin identity and destroyed last by that same identity. After it exists, the Makefile profile assumes it with a short session. The root account's controls are manual checks the owner does by hand.

### Hardening

An SSM association targets instances tagged for hardening and runs the playbook on a schedule and when an instance first becomes managed. The playbook is the upstream CIS role at a pinned tag, Level 1 only, with the overrides in `docs/HARDENING-LINUX.md` (including the custom MOTD, skipped controls 1.8 and 4.6.6, and PAM left unchanged). Every override, skip and exception is in the deviation register in that guideline. The upstream role needs a newer ansible-core than the node ships, and it does not converge in one run, so the association is designed for repeated runs until nothing changes.

### Podium

Podium runs as a single-replica Deployment with a PersistentVolumeClaim mounted at `/data`. Settings come from the chart values:

- `PODIUM_ALLOWED_HOSTS` is the hostname variable.
- `PODIUM_TLS` is on, so the reporter port exists.
- `PODIUM_TRUST_PROXY` is the number of proxy hops in front of the pod. P6.3 confirms the value from Podium's Admin log.
- `PODIUM_TOKEN` comes from a Kubernetes Secret that Terraform creates from a generated random value.

Traefik is configured so server-sent events stay open.

### Secrets

| Secret | Created by | Where it lives | Seen by Claude |
|---|---|---|---|
| Podium token | Terraform (random value) | Kubernetes Secret, and Terraform state | No |
| SSH private key | The Ansible key pair playbook, written locally at mode 0400 | The operator's workstation only | No |
| AWS credentials | The user configures a named profile with short-lived credentials | `~/.aws`, never in a variable | No |

The Podium token is in Terraform state, so the state bucket is encrypted, versioned and not public. `scripts/get-podium-token.sh` reads the Secret with `kubectl` and writes `.secrets/podium-token` with mode `0600`.

### Agent-safe execution

Claude Code runs Terraform, Ansible and cluster commands only through `make` targets. Project permission rules allow those targets and deny reading `~/.aws`, `*.tfstate*`, real `*.tfvars`, `.secrets/`, `.env*` and the SSH key, and deny the commands that print state or secrets (`terraform show`, `terraform state`, `terraform output`, `kubectl get secret`). Targets log to files and never print secrets. The rules are in the repo, so they are reviewable.

### The DNS agent and skill

The agent decides whether a DNS change is needed and why, checks that the load balancer address exists, and asks the skill to make the change. The skill performs the change through the Cloudflare MCP server and then confirms it with a DNS lookup. Guardrails:

- It may create, update or delete only the record named by `podium_hostname`, in the zone for `edengelking.com`.
- The record is always DNS-only.
- It never reads or writes any other record, zone or Cloudflare setting.
- It reads the NLB address from the cluster, and does not accept an address typed into a prompt.
- Every change is followed by a lookup, and the result is reported.

## 5. Configuration

All inputs are Terraform variables with a `terraform.tfvars.example` that is safe to commit. Real `*.tfvars` are gitignored.

| Variable | Meaning |
|---|---|
| `aws_region` | Defaults to `us-west-2`. |
| `aws_profile` | The named profile to use. |
| `name` | Prefix for every resource, so two environments never collide. |
| `podium_hostname` | `demo.edengelking.com`. |
| `podium_image` | A GHCR image reference. It must be `linux/amd64`. No default. |
| `letsencrypt_env` | `staging` or `prod`. Defaults to `staging`. |
| `kubernetes_version` | Defaults to `1.37`. |
| `node_instance_type` | Defaults to `c6i.xlarge`. |
| `operator_cidr` | The operator's public address as a /32, for the EKS public endpoint only. Required, no default, never committed. A changed address needs an apply. |
| `ssh_key_name` | The name of the EC2 key pair the Ansible playbook creates. |
| `gitops_revision` | The branch Argo CD tracks. Defaults to `development`. |
| `enable_*` | One switch per paid security service, default on. `enable_macie` defaults off. |
| `budget_limit_usd`, `budget_email` | The budget alarm. The default of 75 is an estimate for a day or two of running. |

## 6. Feature flags

None.

## 7. Phased build plan

Phases run in order. Each ends in a state someone can observe. Tasks inside a phase that have no shared dependency can be worked in parallel by separate agents, each on its own issue branch. Where a task says "per the guideline", the named sections of `docs/HARDENING-AWS.md` or `docs/HARDENING-LINUX.md` define the controls, and any control not implemented is a deviation recorded with its reason.

### Phase 1 — Foundations

**Goal:** the repo can validate itself, runs Terraform safely, has an image to deploy and has remote state to build on.

#### P1.1 Repo skeleton and tooling checks
- **Scope:** top-level `Makefile`, `scripts/preflight.sh`, `terraform.tfvars.example` files, `.secrets/` gitignore entry. No `terraform/` resources.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `scripts/preflight.sh` exits non-zero and names the missing tool when `terraform`, `ansible`, `aws`, `kubectl`, `helm` or the Session Manager plugin is absent.
  2. `scripts/preflight.sh` exits non-zero when `aws sts get-caller-identity` fails for the configured profile.
  3. `make help` lists one target per stage.
  4. `git check-ignore .secrets/podium-token` succeeds.

#### P1.2 Agent-safe execution
- **Scope:** `.claude/settings.json` permission rules, and the `make` targets that wrap Terraform, Ansible and cluster commands. No `terraform/` resources.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. A request from Claude to read `~/.aws/credentials`, a `*.tfstate` file, a real `*.tfvars`, `.secrets/`, `.env` or the SSH private key is refused by a permission rule.
  2. Requests for `terraform show`, `terraform state`, `terraform output` and `kubectl get secret` are refused.
  3. `make plan-<stage>` runs a Terraform plan and writes its output to `logs/`, and the output contains no value marked sensitive.
  4. `make apply-<stage>` and `make destroy-<stage>` are the only apply and destroy paths, and each still requires a person to approve the command.
  5. The rules are listed in `docs/DECISIONS.md` with the reason for each.

#### P1.3 Remote state bucket
- **Scope:** `terraform/state/` only.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. `terraform apply` creates a private, versioned, encrypted bucket with public access blocked.
  2. The other roots can configure it as a backend with `use_lockfile = true`, and a second concurrent `plan` fails with a lock error.
  3. The bucket meets the S3 controls for the state bucket in `docs/HARDENING-AWS.md` section 5.8, or records the deviation.
  4. It is removed, including versions, by teardown.

#### P1.4 Podium image build and publish
- **Scope:** `scripts/build-image.sh` and the image reference in the example tfvars. Nothing in the Podium source repo is changed.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. The script builds the Podium image for `linux/amd64` and pushes it to GHCR as a public package, tagged with the Podium version.
  2. An image inspection of the pushed tag shows `linux/amd64`.
  3. A pull of the image works without logging in.
  4. The script prints no token, and uses an existing `gh` login.

**Exit criterion:** `make preflight` passes, a refused-read test for each denied path passes, the state bucket holds a lock file during a plan, and the image can be pulled anonymously.

### Phase 2 — Account baseline and deployer role

**Goal:** the account has its hardening baseline and a scoped identity to build with.
**Needs from Phase 1:** the state bucket (P1.3) and the safe `make` targets (P1.2).

#### P2.1 Deployer and operator roles
- **Scope:** the deployer role, the `operator-ssh` role and their policies in `terraform/account/`, and the `make` profile that assumes the deployer role.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. An admin identity creates the roles once. The deployer policy grants only the service and action groups in `docs/HARDENING-AWS.md` section 5.12, and contains no `*:*`.
  2. The `operator-ssh` role can start Session Manager sessions only against the tagged node and only with the SSH document. The deployer role is not widened for this.
  3. The Makefile profile assumes the deployer role with a short session, and `aws sts get-caller-identity` shows the role.
  4. The deployer role cannot create an IAM user, attach an administrator policy or change the CloudTrail or Config settings it does not own.
  5. Destroying the roles is the last step of teardown and is done by the admin identity, not by the roles.

#### P2.2 Encryption keys and account settings
- **Scope:** KMS keys, the IAM password policy, the S3 account public access block and EBS default encryption, in `terraform/account/`.
- **Depends on (in-phase):** P2.1
- **Acceptance:**
  1. Each setting meets its control per the guideline (account and IAM, encryption and KMS), or is a recorded deviation.
  2. KMS keys use the shortest deletion window, and key rotation is on.
  3. The three account settings are reset by teardown and are checked by the teardown validation.

#### P2.3 Logging and threat detection
- **Scope:** CloudTrail (management and S3 data events), Config, Security Hub with the CIS standard, GuardDuty, Inspector, in `terraform/account/`. Macie is not created.
- **Depends on (in-phase):** P2.2
- **Acceptance:**
  1. Each service is enabled per the guideline, with its `enable_*` switch on by default.
  2. Log buckets are encrypted, versioned, not public and use `force_destroy` so teardown can remove them.
  3. Controls that need an AWS Organization are recorded as not applicable.

#### P2.4 Cost guard
- **Scope:** the AWS budget alarm in `terraform/account/`, and logging in the `make` targets.
- **Depends on (in-phase):** P2.1
- **Acceptance:**
  1. A budget with an email alert exists, and its default limit is the estimate in section 5.
  2. Every `make` target writes its output to a file under `logs/`, and none writes a credential or token to it.

**Exit criterion:** the account root applies cleanly with the deployer role, the services report enabled, and a destroy and re-apply of the root works.

### Phase 3 — Network and cluster

**Goal:** an empty, hardened EKS cluster.
**Needs from Phase 2:** the deployer role (P2.1), the KMS keys (P2.2) and logging (P2.3).

#### P3.1 Network
- **Scope:** `terraform/network/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. A VPC with public and private subnets across at least two availability zones, tagged for EKS and load balancers, meeting the networking controls in the AWS guideline, including flow logs and the default security group.
  2. Interface endpoints for the Systems Manager services (`ssm`, `ssmmessages`, `ec2messages`) exist and are used by the node.
  3. Outputs expose VPC ID and subnet IDs, and the cluster root reads them only through remote state.
  4. `terraform destroy` leaves no VPC, subnet, NAT gateway, endpoint or elastic IP tagged with `name`.

#### P3.2 Cluster and node
- **Scope:** `terraform/cluster/` EKS, node group, launch template, IAM roles. Not the add-ons and not the SSM association.
- **Depends on (in-phase):** P3.1
- **Acceptance:**
  1. The cluster runs `kubernetes_version`. If EKS does not offer it, the plan fails with a message naming the versions it does offer.
  2. The public endpoint is limited to `operator_cidr` and the private endpoint is on. Secrets encryption, control plane logging and the other EKS controls are applied per the guideline.
  3. A managed node group with one Amazon Linux 2023 amd64 node joins, and `kubectl get nodes` shows it `Ready`. The launch template requires IMDSv2 and encrypts its volume.
  4. The node security group has no inbound SSH rule from any source. The launch template names `ssh_key_name`.
  5. The node carries the tag the hardening association targets, and has the SSM managed-instance policy attached.
  6. `aws eks update-kubeconfig` produces a working context, and the script that does it does not print credentials.
  7. With Argo CD, Traefik, cert-manager and Podium running, `kubectl top node` shows memory still free.

#### P3.3 EBS CSI driver and StorageClass
- **Scope:** the EBS CSI add-on and its IAM role (Pod Identity or IRSA) in `terraform/cluster/`, plus the `gp3` StorageClass.
- **Depends on (in-phase):** P3.2
- **Acceptance:**
  1. A test PVC against the `gp3` StorageClass binds, and a pod writes and reads a file on it.
  2. The CSI controller uses a dedicated role with only the volume permissions it needs.

**Exit criterion:** `kubectl get nodes` shows a `Ready` node, a throwaway PVC binds and is writable, and the EKS controls that the Prowler and kube-bench checks can run pass or are recorded as deviations.

### Phase 4 — Node hardening and access

**Goal:** the node is hardened, replacements are hardened automatically, and the operator can SSH in with no open port.
**Needs from Phase 3:** the node group and its tag (P3.2) and the Systems Manager endpoints (P3.1).

#### P4.1 SSH key pair
- **Scope:** `ansible/playbooks/` key pair playbook only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The playbook creates the EC2 key pair named by `ssh_key_name` with `amazon.aws.ec2_key` (key type `ed25519` or `rsa`) and writes the private key to the operator's workstation at mode `0400`, with `no_log` set. The mode is set by a separate task and tested.
  2. The key never appears in Terraform state, in a log or on the terminal.
  3. Running it again when the key pair exists does not overwrite the key or fail.
  4. A second playbook run with `state: absent` deletes the key pair, and teardown uses it.

#### P4.2 Hardening role
- **Scope:** `ansible/roles/` overrides around the upstream CIS Level 1 role, pinned to a tag, and the playbook that calls it. No Terraform.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The role applies the controls in `docs/HARDENING-LINUX.md` section 5, with every override, skip and exception from the deviation register (7.6), including the custom MOTD.
  2. Level 2 controls are switched off using Appendix A.
  3. Running the playbook repeatedly converges: the number of runs to reach zero changes is recorded, and a final run reports no changes.
  4. The role records each applied control by ID in a file on the node.
  5. `ansible-lint` and `ansible-playbook --syntax-check` pass.

#### P4.3 SSM association
- **Scope:** the S3 object for the playbook and the SSM association in `terraform/cluster/` (a separate file from P3.2's).
- **Depends on (in-phase):** P4.2
- **Acceptance:**
  1. The association targets nodes by tag and runs on a schedule and when a node first registers. It makes a suitable ansible-core available on the node.
  2. After apply, the node's association status is `Success`.
  3. Terminating the node results in a replacement that reports `Success` without anyone running a command. With one node this is a short outage of everything on the cluster, and the test is run only before Podium is deployed.

#### P4.4 SSH over Session Manager
- **Scope:** the sshd drop-in and Instance Connect removal in the hardening role, the operator-side SSH configuration documented in the README, and the use of the `operator-ssh` role. No Terraform resources for the node.
- **Depends on (in-phase):** P4.1, P4.3
- **Acceptance:**
  1. `ssh` to the node using the downloaded key and the Session Manager `ProxyCommand` succeeds.
  2. A connection attempt to the node's address on port 22 from the outside fails, because no inbound rule exists.
  3. Password authentication and logins as any user but the allowed one are refused, per the SSH rows in the Linux guideline.
  4. Shell-session logging is on as the AWS guideline requires. SSH sessions are not covered by Session Manager's own session logging, so the evidence for them is the CloudTrail `StartSession` event, and a test confirms that event is recorded.

#### P4.5 Hardening verification
- **Scope:** `scripts/verify-hardening.sh`, which calls the validation tool for the node controls once the tool exists, and runs the node-side checks directly until then.
- **Depends on (in-phase):** P4.3
- **Acceptance:**
  1. The script checks each applied control on the node and prints a list of control IDs with pass or fail. It prints no percentage.
  2. It exits non-zero if any applied control fails.

**Exit criterion:** `scripts/verify-hardening.sh` passes, passes again after the node is terminated and replaced, and the SSH tunnel works.

### Phase 5 — GitOps, add-ons and cluster visibility

**Goal:** Argo CD, ingress, certificates and a read-only view of the cluster work, with nothing deployed on top yet.
**Needs from Phase 3:** a working cluster and the `gp3` StorageClass (P3.3).

#### P5.1 Argo CD bootstrap
- **Scope:** `terraform/addons/` Argo CD release and the root app-of-apps Application. `gitops/apps/` root only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Argo CD is running, and the root Application syncs from this repo at `gitops_revision`.
  2. The Argo CD admin secret is not printed by any script and is not in Git.
  3. Sync policy is automated with prune and self-heal, and a manual change to a managed resource is reverted.

#### P5.2 Traefik, load balancer and the 4318 passthrough
- **Scope:** an Argo CD Application for Traefik under `gitops/apps/`, including the NLB listeners and the TCP entrypoint, and a throwaway TLS echo service used for the test.
- **Depends on (in-phase):** P5.1
- **Acceptance:**
  1. One NLB is provisioned with listeners on 80, 443 and 4318, and Traefik reports ready.
  2. Before Podium is deployed, a stand-in service that requires a client certificate sits behind the 4318 passthrough. A connection without a client certificate is refused, and a connection with a valid one succeeds. If this fails, the design is revisited before anything else is built on it.
  3. Traefik is configured per the ingress rows in `docs/HARDENING-AWS.md` section 5.9, including TLS versions, hidden version, no exposed dashboard, access logs and timeouts that keep server-sent events open.
  4. The chart's handling of the custom entrypoint on the Service is confirmed and recorded.

#### P5.3 cert-manager and issuers
- **Scope:** an Argo CD Application for cert-manager and the `ClusterIssuer` for staging and prod under `gitops/`, both using the HTTP-01 solver through Traefik.
- **Depends on (in-phase):** P5.1, P5.2
- **Acceptance:**
  1. Both issuers exist and show `Ready` with an ACME account registered, and the active one is chosen by `letsencrypt_env` alone.
  2. The solver needs no credential, and no Secret in the cluster holds a Cloudflare token.
  3. Port 80 on the NLB reaches Traefik, so a challenge request for `/.well-known/acme-challenge/` can be served.

#### P5.4 Kubernetes MCP server
- **Scope:** the MCP server entry in `.mcp.json`, plus the service account, custom role and kubeconfig it uses. Nothing else in the cluster.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The server is `containers/kubernetes-mcp-server`, pinned to an exact version and not `@latest`.
  2. It runs with `--config` pointing at a TOML file in the repo that sets `read_only = true`, limits `toolsets`, and lists `Secret` under `denied_resources`. The file holds no secret.
  3. It authenticates with a kubeconfig for a dedicated service account, using a short-lived token, passed through `KUBECONFIG`. The kubeconfig is gitignored.
  4. The role grants only get, list and watch on `Certificate`, `CertificateRequest`, `Order`, `Challenge`, `ClusterIssuer`, and on pods and events in the add-on namespaces. It is not the built-in `view` or `cluster-reader` role.
  5. A request to read a Secret, or to create, change or delete anything, is refused. Each protection is tested alone.
  6. An agent can use it to report why a certificate is not `Ready`.

#### P5.5 In-cluster controls
- **Scope:** namespace labels, network policies and the other GitOps-managed rows in `docs/HARDENING-AWS.md` section 5.5 under `gitops/`.
- **Depends on (in-phase):** P5.1
- **Acceptance:**
  1. Every namespace has a default-deny network policy and the allows the add-ons need.
  2. Pod Security Admission labels are set per the guideline, and each chart runs under its labeled level or its deviation is recorded.
  3. kube-bench's in-cluster checks pass or are recorded as deviations.

**Exit criterion:** both issuers are `Ready`, the NLB has an address, the 4318 passthrough test passes, and the Kubernetes MCP server reports the issuers' state.

### Phase 6 — Podium and DNS

**Goal:** Podium answers on the public hostname, and reporters can connect to the reporter port.
**Needs from Phase 5:** Argo CD (P5.1), Traefik (P5.2), the issuers (P5.3) and the Kubernetes MCP server (P5.4).

#### P6.1 Podium Helm chart
- **Scope:** `gitops/charts/podium/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `helm lint` and `helm template` pass with the example values.
  2. The chart renders one replica, a PVC mounted at `/data`, a Service on 4317 and 4318, an Ingress and an `IngressRouteTCP` for the hostname, and a Certificate for it.
  3. `PODIUM_ALLOWED_HOSTS`, `PODIUM_TLS` and `PODIUM_TRUST_PROXY` come from values. `PODIUM_TOKEN` comes from a Secret reference and never from values.
  4. The image reference has no default, and rendering without it fails with a named error.
  5. The pod runs as a non-root user with a read-only root filesystem where Podium allows it, per the guideline.

#### P6.2 Token generation and retrieval
- **Scope:** the generated token and Secret in `terraform/addons/`, and `scripts/get-podium-token.sh`.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Terraform creates the Secret in the Podium namespace from a generated random value, and the value is not in any file in the repo.
  2. `scripts/get-podium-token.sh` writes `.secrets/podium-token` with mode `0600` and prints no token to the terminal.
  3. The token is not visible in the Argo CD UI.

#### P6.3 Podium application
- **Scope:** the Argo CD Application that points at the chart.
- **Depends on (in-phase):** P6.1, P6.2
- **Acceptance:**
  1. One variable, `podium_hostname`, sets the Ingress host, the certificate name and `PODIUM_ALLOWED_HOSTS`.
  2. Podium reaches `Ready`, and the PVC is bound.
  3. The Admin log shows the real client address as the source for a console request, and `PODIUM_TRUST_PROXY` is set to the value that makes that true.
  4. Where Podium keeps its TLS CA in the pod is confirmed, and the CA survives a restart on the volume.

#### P6.4 DNS agent and skill
- **Scope:** `.claude/agents/` and `.claude/skills/` for the DNS work. No Terraform, no cluster resources.
- **Depends on (in-phase):** none in this phase. Needs P5.2 (the NLB address) and P5.4 (the Kubernetes MCP server) from Phase 5.
- **Acceptance:**
  1. The agent's tool list contains only the Cloudflare MCP tools, the Kubernetes MCP tools and read-only shell lookups. It has no Terraform and no file-writing tools.
  2. Asked to point `demo.edengelking.com` at the NLB, it reads the NLB address from the cluster, makes the change through the skill, and confirms it with a DNS lookup.
  3. Asked to change any other record, zone or setting, it refuses and says why.
  4. Asked to use an address typed into the prompt, it refuses and uses the cluster's.
  5. The record it creates is DNS-only.
  6. Running it again when the record is already correct makes no change and says so.

#### P6.5 Certificate and reporter port
- **Scope:** the production certificate, the reporter port, and the checks for both. No new components.
- **Depends on (in-phase):** P6.3, P6.4
- **Acceptance:**
  1. With the DNS record in place and the staging issuer, the `Certificate` reaches `Ready`, and `curl` to `https://demo.edengelking.com/` returns Podium's page with the staging certificate.
  2. With the prod issuer, the browser shows a trusted certificate. The agent uses the Kubernetes MCP server to confirm the `Certificate` is `Ready` before and after the switch, and to report the failing `Challenge` if it is not.
  3. A reporter config created in Admin > People from the public address connects over 4318, and a Podium simulator session appears on the wall.
  4. A connection to 4318 without a client certificate is refused.

**Exit criterion:** `https://demo.edengelking.com/` serves Podium with a valid certificate, and the Podium simulator can report a session to it over the reporter port.

### Phase 7 — Validation and operability

**Goal:** the hardening can be checked, the whole thing can be run end to end, and it can be removed and proven removed.
**Needs from Phase 6:** a working deployment to check and tear down. P7.1 and P7.2 can start earlier, because they need only the guidelines.

#### P7.1 Control lists
- **Scope:** `hardening/controls/` only: the machine-readable YAML for the AWS controls and the Linux controls, in the schema in `docs/HARDENING-AWS.md` section 10.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Every control row in both guidelines is an entry with its ID, source, level, verification method and expected result, or is listed as a deviation with its reason.
  2. A schema check validates every file, and rejects an entry with a missing field.
  3. No entry contains a credential, account ID or ARN.

#### P7.2 Validation tool
- **Scope:** `hardening/tool/` only: the Python tool, its tests and fixtures.
- **Depends on (in-phase):** P7.1
- **Acceptance:**
  1. The tool reads Ansible output, Prowler output and kube-bench output, maps them to control IDs and prints one line per control: pass, fail, not applicable or manual, with the evidence.
  2. It prints no percentage, and exits non-zero if any control that applies fails.
  3. Tests with fixtures cover a pass, a fail, a not-applicable, a manual control and a missing result.
  4. A redaction test shows it never prints a credential, token, account ID or ARN.
  5. Deviations in the control list are reported as deviations, not as failures.

#### P7.3 End-to-end verify script
- **Scope:** `scripts/verify.sh` only.
- **Depends on (in-phase):** P7.2
- **Acceptance:**
  1. It runs the validation tool and checks the node, Argo CD sync status, the certificate, HTTPS, the reporter port and the Podium health response, printing one line per check.
  2. It holds a server-sent events connection open for ten minutes and fails if it drops.
  3. It exits non-zero if any check fails.

#### P7.4 Teardown
- **Scope:** `scripts/teardown.sh`, and the order in which roots are destroyed.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The script asks for the name of the environment before destroying and does nothing on a mismatch.
  2. It stops with a message if `demo.edengelking.com` still resolves to the NLB, telling the user to ask the DNS agent to remove the record first.
  3. It deletes the key pair with the key pair playbook, then destroys add-ons, cluster, network, the account root and the state bucket, and the deployer and operator roles last, using the admin identity.
  4. Resources Kubernetes created outside Terraform (load balancers, volumes, network interfaces) are removed before the network is destroyed.
  5. A second run on an already-clean account exits zero.

#### P7.5 Teardown validation
- **Scope:** `scripts/verify-teardown.sh` only, per the design in `docs/HARDENING-AWS.md` section 9.
- **Depends on (in-phase):** P7.4
- **Acceptance:**
  1. One check per resource type, each with the query that proves it is gone, and one output line per check with pass or fail. It prints no account IDs or ARNs.
  2. It finds resources Terraform did not create by tag, name prefix and service-specific listings.
  3. A KMS key in pending deletion counts as clean. The three account settings, the key pair and the load balancers each have a check.
  4. It works after the deployer role has been destroyed.
  5. After a full teardown it exits zero, and with a deliberately left-behind resource it exits non-zero and names the check.

#### P7.6 Documentation
- **Scope:** `README.md`, `docs/ARCHITECTURE.md`, `docs/DECISIONS.md`.
- **Depends on (in-phase):** P7.3, P7.5
- **Acceptance:**
  1. The README has a quick start that a person can follow from an empty account, the SSH configuration for the Session Manager tunnel, and a section on rate limits for the production issuer.
  2. `docs/DECISIONS.md` has an entry for each decision named in this spec and in the guidelines' deviation tables.
  3. `docs/ARCHITECTURE.md` matches what was built, and says where it departs from this spec.

**Exit criterion:** from an empty account, the documented sequence reaches a passing `scripts/verify.sh`, and `scripts/teardown.sh` followed by `scripts/verify-teardown.sh` returns the account to empty.

## 8. Explicitly out of scope / deferred

- An access layer in front of the Podium console. The design must not preclude one, since Traefik is the single entry point for the console.
- Moving Podium to another database and more than one replica, when Podium supports it.
- A baked hardened image in place of the SSM association. The role is written so it can be reused in an image build.
- CIS Level 2 on the node.
- Policy checks on `terraform plan` in CI (security, cost, tagging).
- Cloudflare's proxy in front of the console. Possible later with a second hostname for the reporter port.
- Newer benchmark versions: CIS AWS Foundations 7.1.0 and CIS EKS 2.0.0. The guidelines use the numbering that public sources provide (7.0.0 and 1.8.0).

## 9. Open questions

1. **Traefik on 4318.** The passthrough keeps Podium's mutual TLS intact by inference, not by documentation. P5.2 tests it first, before anything is built on it.
2. **Kubernetes MCP server settings.** The option names come from the project's README as read on 2026-10-24. P5.4 re-checks them against the pinned version.
3. **Upstream role convergence.** The pinned role does not reach zero changes in one run, and the node's ansible-core is older than the role needs. P4.2 and P4.3 settle both.
4. **Unverified guideline lines.** `docs/HARDENING-AWS.md` and `docs/HARDENING-LINUX.md` mark lines they could not check against a source as UNVERIFIED. Each is resolved or accepted as the matching task is built.
5. **Budget.** The $75 default is an estimate from recalled prices and is checked against AWS pricing before the first apply.
