# SPEC: EKS platform bootstrap for Podium

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-24 | First draft. Provisional until the open questions in section 9 are answered. |

This project stands up a hardened Kubernetes platform on AWS from nothing and deploys [Podium](https://github.com/eengelking/podium) onto it. Terraform creates the network and an EKS cluster. Ansible hardens the nodes and keeps them hardened as they are replaced. Argo CD is bootstrapped and deploys Podium from a Helm chart in this repo. Cloudflare DNS points a hostname at the cluster, and cert-manager issues a Let's Encrypt certificate for it.

One person runs it from a workstation with short-lived AWS credentials. The repo is public, and its history is meant to be read, so every stage is a separate, named, testable unit.

## 1. Goals and non-goals

### Goals

- One documented command sequence takes an empty AWS account region to Podium answering over HTTPS on a public hostname.
- Every stage (state bootstrap, network, cluster, add-ons, hardening, Podium) can be run, verified and torn down on its own.
- Nodes are hardened after they join, and replaced nodes are re-hardened without anyone running anything.
- The Let's Encrypt environment (`staging` or `prod`) is one variable.
- The public hostname is one variable. The DNS record and Podium's host and proxy settings are all derived from it.
- Teardown removes everything the tool created and is tested before anything else depends on it.
- No secret is ever in the repo, in Argo CD's UI or in a log.

### Non-goals (do NOT build these)

- High availability for Podium. It runs one replica on a SQLite volume. Other databases and multiple replicas are Podium's roadmap, not this tool's.
- Multi-region, multi-cluster or multi-account support.
- A pre-baked hardened node image (Packer). It is the better production pattern and is documented in `docs/DECISIONS.md`, not built.
- Bottlerocket or any node OS other than Amazon Linux 2023.
- A compliance score. The tool names the benchmark it applied and the controls it applied. It does not report a pass rate.
- Auto-approving or auto-applying Terraform to a real account. Every apply and destroy is run by a person.
- Reporting real Claude Code sessions to the deployed Podium. Only the Podium simulator reports to it.
- Authentication in front of Podium. Reads are open by design on a trusted network. Putting an access layer in front is deferred (section 8).
- Building or publishing the Podium image. The tool takes an image reference as input.
- Managing Cloudflare account-level settings beyond one DNS record and its proxy mode.

## 2. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Infrastructure | Terraform 1.17 | State, plan review and drift detection. Separate roots per concern keep a destroy of one from touching the others. |
| State | S3 bucket with native locking (`use_lockfile`) | No DynamoDB table to create and tear down. |
| Region and arch | `us-west-2`, amd64 | Chosen by the maintainer. |
| Kubernetes | EKS, managed node group, current standard-support version | Managed upgrades and node replacement. |
| Node OS | Amazon Linux 2023 EKS-optimized AMI | SSM agent is present, and a CIS benchmark exists for it. |
| Hardening | Ansible, applied through an SSM association | Re-applies to nodes the group replaces. A one-time run would be lost on the next recycle. |
| Storage | EBS CSI driver add-on, `gp3` StorageClass | Podium's SQLite volume needs a PersistentVolumeClaim. |
| GitOps | Argo CD, app-of-apps | One bootstrap app owns the rest, and everything after it is a Git change. |
| Ingress | ingress-nginx behind an NLB | Works with cert-manager DNS-01, and buffering and timeouts are configurable for Podium's server-sent events. |
| Certificates | cert-manager, DNS-01 solver, one `staging`/`prod` variable | DNS-01 works with a proxied Cloudflare record. |
| DNS | Cloudflare, proxied record, SSL mode Full (strict) | See section 9, question 2 for how the record is managed. |
| App packaging | A small Helm chart in this repo | Argo CD renders it. Values carry the hostname, image and settings. |
| Scripting | Bash, `make` | One script or target per stage. |

## 3. Glossary

- **Stage:** one independently runnable unit of the build: state, network, cluster, add-ons, hardening, Podium.
- **Root:** a Terraform root module with its own state, one per stage that Terraform owns.
- **Issuer variable:** `letsencrypt_env`, either `staging` or `prod`.
- **Hostname variable:** `podium_hostname`, the one source for the DNS record, the certificate name and Podium's allowed hosts.

## 4. Architecture

### Layout

```
terraform/
  state/        S3 bucket for remote state (local state, created once)
  network/      VPC, subnets, NAT
  cluster/      EKS, node group, IAM, EBS CSI add-on, SSM association
  addons/       Argo CD, ingress-nginx, cert-manager, issuers, Podium secret, DNS
ansible/
  roles/        node hardening role(s)
  playbooks/    the playbook the SSM association runs
gitops/
  apps/         Argo CD Application manifests (app-of-apps)
  charts/podium/ Helm chart for Podium
scripts/        one script per stage, plus verify, teardown, token
```

`network`, `cluster` and `addons` read each other's outputs through remote state. A stage never reads another stage's files.

### Hardening

An SSM association targets instances tagged for hardening and runs the playbook on a schedule and when an instance first becomes managed. The playbook is uploaded to S3 by Terraform. The role applies selected Level 1 server controls from the CIS Amazon Linux 2023 Benchmark and lists each applied control by ID. Controls that would break Kubernetes (for example disabling IP forwarding) are skipped and listed as deliberate deviations in `docs/DECISIONS.md`.

### Podium

Podium runs as a single-replica Deployment with a PersistentVolumeClaim mounted at `/data`. It listens on 4317 (and exposes 4318). Settings come from the chart values:

- `PODIUM_ALLOWED_HOSTS` is the hostname variable.
- `PODIUM_TRUST_PROXY` is set to the number of proxy hops in front of the pod, and only a proxy that overwrites `X-Forwarded-For` counts as a hop.
- `PODIUM_TOKEN` comes from a Kubernetes Secret that Terraform creates from a generated random value.

The ingress sets proxy buffering off and long read and send timeouts, because the browser view streams over server-sent events.

### Secrets

The Podium token is generated by Terraform and stored in a Secret that Argo CD does not manage. `scripts/get-podium-token.sh` reads it with `kubectl` and writes it to `.secrets/podium-token` with mode `0600`. The token is in Terraform state, so the state bucket is encrypted, versioned and not public. This is an accepted risk, recorded in `docs/DECISIONS.md`.

## 5. Configuration

All inputs are Terraform variables with a `terraform.tfvars.example` that is safe to commit. Real `*.tfvars` are gitignored.

| Variable | Meaning |
|---|---|
| `aws_region` | Defaults to `us-west-2`. |
| `name` | Prefix for every resource, so two environments never collide. |
| `podium_hostname` | The public name. |
| `podium_image` | Image reference. No default. It must be `linux/amd64`. |
| `letsencrypt_env` | `staging` or `prod`. Defaults to `staging`. |
| `cloudflare_zone` | The zone that holds the hostname. |
| `budget_limit_usd`, `budget_email` | The budget alarm. |

AWS and Cloudflare credentials come from the environment and are never variables.

## 6. Feature flags

None.

## 7. Phased build plan

Phases run in order. Each ends in a state someone can observe. Tasks inside a phase that have no shared dependency can be worked in parallel by separate agents, each on its own issue branch.

### Phase 1 — Foundations

**Goal:** the repo can validate itself and has remote state to build on.

#### P1.1 Repo skeleton and tooling checks
- **Scope:** top-level `Makefile`, `scripts/preflight.sh`, `terraform.tfvars.example` files, `.secrets/` gitignore entry. Nothing under `terraform/` resources.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `scripts/preflight.sh` exits non-zero and names the missing tool when `terraform`, `ansible`, `aws`, `kubectl` or `helm` is absent.
  2. `scripts/preflight.sh` exits non-zero when `aws sts get-caller-identity` fails.
  3. `make help` lists one target per stage.
  4. `git check-ignore .secrets/podium-token` succeeds.

#### P1.2 Remote state bucket
- **Scope:** `terraform/state/` only.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. `terraform apply` in `terraform/state/` creates a private, versioned, encrypted bucket with public access blocked.
  2. The other roots can configure it as a backend with `use_lockfile = true`, and a second concurrent `plan` fails with a lock error.
  3. `terraform destroy` in `terraform/state/` removes the bucket, including versions, when run through the teardown script.

**Exit criterion:** `make preflight` passes and the state bucket exists and holds a lock file during a plan.

### Phase 2 — Network and cluster

**Goal:** an empty, working EKS cluster.
**Needs from Phase 1:** the state bucket (P1.2) and `make` targets (P1.1).

#### P2.1 Network
- **Scope:** `terraform/network/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `terraform plan` shows a VPC with public and private subnets across at least two availability zones, tagged for EKS and load balancers.
  2. Outputs expose VPC ID and subnet IDs, and the cluster root reads them only through remote state.
  3. `terraform destroy` leaves no VPC, subnet, NAT gateway or elastic IP tagged with `name`.

#### P2.2 Cluster and node group
- **Scope:** `terraform/cluster/` EKS, node group, IAM roles. Not the add-ons and not the SSM association.
- **Depends on (in-phase):** P2.1
- **Acceptance:**
  1. A managed node group of Amazon Linux 2023 amd64 nodes joins, and `kubectl get nodes` shows all of them `Ready`.
  2. Nodes carry the tag the hardening association targets, and have the SSM managed-instance policy attached.
  3. `aws eks update-kubeconfig` produces a working context, and the script that does it does not print credentials.

#### P2.3 EBS CSI driver and StorageClass
- **Scope:** the EBS CSI add-on and its IAM role (Pod Identity or IRSA) in `terraform/cluster/`, plus the `gp3` StorageClass.
- **Depends on (in-phase):** P2.2
- **Acceptance:**
  1. A test PVC against the `gp3` StorageClass binds, and a pod writes and reads a file on it.
  2. The CSI controller uses a dedicated role with only the volume permissions it needs.

**Exit criterion:** `kubectl get nodes` shows `Ready` nodes and a throwaway PVC binds and is writable.

### Phase 3 — Node hardening

**Goal:** nodes are hardened, and replacements are hardened automatically.
**Needs from Phase 2:** the node group and its tag (P2.2).

#### P3.1 Hardening role
- **Scope:** `ansible/roles/` and `ansible/playbooks/`. No Terraform.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The role applies the chosen controls and records each by CIS control ID in a file on the node.
  2. Running the playbook twice reports no changes the second time.
  3. Deliberate deviations (IP forwarding and any other control that breaks Kubernetes) are listed in the role's README and `docs/DECISIONS.md`.
  4. `ansible-lint` and `ansible-playbook --syntax-check` pass.

#### P3.2 SSM association
- **Scope:** the S3 object for the playbook and the SSM association in `terraform/cluster/` (a separate file from P2.2's).
- **Depends on (in-phase):** P3.1
- **Acceptance:**
  1. The association targets nodes by tag and runs on a schedule and when a node first registers.
  2. After apply, every node's association status is `Success`.
  3. Terminating one node in the group results in a replacement that reports `Success` without anyone running a command.

#### P3.3 Hardening verification script
- **Scope:** `scripts/verify-hardening.sh` only.
- **Depends on (in-phase):** P3.2
- **Acceptance:**
  1. The script checks each applied control on every node and prints a list of control IDs with pass or fail. It prints no percentage.
  2. It exits non-zero if any applied control fails.

**Exit criterion:** `scripts/verify-hardening.sh` passes, then passes again after one node is terminated and replaced.

### Phase 4 — GitOps and add-ons

**Goal:** Argo CD, ingress and certificates work, with nothing deployed on top yet.
**Needs from Phase 2:** a working cluster and the `gp3` StorageClass (P2.3).

#### P4.1 Argo CD bootstrap
- **Scope:** `terraform/addons/` Argo CD release and the root app-of-apps Application. `gitops/apps/` root only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Argo CD is running, and the root Application syncs from this repo at a revision set by a variable that defaults to `main`.
  2. The Argo CD admin secret is not printed by any script and is not in Git.
  3. Sync policy is automated with prune and self-heal, and a manual change to a managed resource is reverted.

#### P4.2 Ingress controller
- **Scope:** an Argo CD Application for ingress-nginx under `gitops/apps/`.
- **Depends on (in-phase):** P4.1
- **Acceptance:**
  1. An NLB is provisioned and the controller reports ready.
  2. The controller configuration sets proxy buffering off and read and send timeouts of at least one hour for the Podium ingress.

#### P4.3 cert-manager and issuers
- **Scope:** an Argo CD Application for cert-manager, and the `ClusterIssuer` for staging and prod under `gitops/`.
- **Depends on (in-phase):** P4.1
- **Acceptance:**
  1. Both issuers exist, and the active one is chosen by `letsencrypt_env` alone.
  2. The DNS-01 solver reads its Cloudflare token from a Secret created outside Git, and the token does not appear in Argo CD.
  3. A test `Certificate` against the staging issuer reaches `Ready` for a name in the zone.

**Exit criterion:** a test `Certificate` for a name in the zone becomes `Ready` through the staging issuer, and the NLB has an address.

### Phase 5 — Podium

**Goal:** Podium answers over HTTPS on the public hostname.
**Needs from Phase 4:** Argo CD (P4.1), the ingress (P4.2) and the issuers (P4.3).

#### P5.1 Podium Helm chart
- **Scope:** `gitops/charts/podium/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `helm lint` and `helm template` pass with the example values.
  2. The chart renders one replica, a PVC mounted at `/data`, a Service on 4317, an Ingress for the hostname and a Certificate for it.
  3. `PODIUM_ALLOWED_HOSTS` and `PODIUM_TRUST_PROXY` come from values. `PODIUM_TOKEN` comes from a Secret reference and never from values.
  4. The image reference has no default, and rendering without it fails with a named error.

#### P5.2 Token generation and retrieval
- **Scope:** the generated token and Secret in `terraform/addons/`, and `scripts/get-podium-token.sh`.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Terraform creates the Secret in the Podium namespace from a generated random value, and the value is not in any file in the repo.
  2. `scripts/get-podium-token.sh` writes `.secrets/podium-token` with mode `0600` and prints no token to the terminal.
  3. The token is not visible in the Argo CD UI.

#### P5.3 Hostname, DNS record and Podium application
- **Scope:** the Cloudflare record in `terraform/addons/`, and the Argo CD Application that points at the chart.
- **Depends on (in-phase):** P5.1, P5.2
- **Acceptance:**
  1. One variable, `podium_hostname`, sets the DNS record, the Ingress host, the certificate name and `PODIUM_ALLOWED_HOSTS`.
  2. The record points at the NLB and is proxied, with SSL mode Full (strict).
  3. With the staging issuer, `curl` to the hostname returns Podium's page and the certificate is the staging one.
  4. With the prod issuer, the browser shows a trusted certificate.
  5. Podium's Admin log shows the real client address as the source for a request, not the proxy's.

**Exit criterion:** `https://<podium_hostname>/` serves Podium with a valid certificate, and the Podium simulator can report a session to it.

### Phase 6 — Operability

**Goal:** the whole thing can be run, checked and removed safely.
**Needs from Phase 5:** a working deployment to verify and tear down.

#### P6.1 End-to-end verify script
- **Scope:** `scripts/verify.sh` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. It checks nodes, Argo CD sync status, the certificate, HTTPS and the Podium health response, and prints one line per check.
  2. It holds a server-sent events connection open for ten minutes and fails if it drops.
  3. It exits non-zero if any check fails.

#### P6.2 Teardown
- **Scope:** `scripts/teardown.sh`, and the order in which roots are destroyed.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The script asks for the name of the environment before destroying and does nothing on a mismatch.
  2. It destroys add-ons, cluster, network and state in that order, and removes the DNS record and load balancer first so none are orphaned.
  3. After a full run, `aws` queries by tag find no VPC, cluster, load balancer, volume, elastic IP or bucket for that name.
  4. A second run on an already-clean account exits zero.

#### P6.3 Cost guard and build logging
- **Scope:** the AWS budget alarm (in `terraform/state/`), and logging in the `make` targets.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. A budget with an email alert exists after the state stage.
  2. Every `make` target writes its output to a file under `logs/`, and none writes a credential or token to it.

#### P6.4 Documentation
- **Scope:** `README.md`, `docs/ARCHITECTURE.md`, `docs/DECISIONS.md`.
- **Depends on (in-phase):** P6.1, P6.2, P6.3
- **Acceptance:**
  1. The README has a quick start that a person can follow from an empty account, and a section on rate limits for the production issuer.
  2. `docs/DECISIONS.md` has an entry for each decision named in this spec.
  3. `docs/ARCHITECTURE.md` matches what was built, and says where it departs from this spec.

**Exit criterion:** from an empty region, the documented sequence reaches a passing `scripts/verify.sh`, and `scripts/teardown.sh` returns the region to empty.

## 8. Explicitly out of scope / deferred

- An access layer in front of Podium. The design must not preclude one, since the ingress is the single entry point.
- Moving Podium to another database and more than one replica, when Podium supports it.
- A baked hardened image in place of the SSM association. The role is written so it can be reused in an image build.
- Policy checks on `terraform plan` in CI (security, cost, tagging).
- Wiring Podium's messaging between orchestrators into the build process.

## 9. Open questions

1. **Podium image.** Where it is published, and how it is built for `linux/amd64`. The local image is arm64 and will not schedule on these nodes. Until decided, `podium_image` has no default.
2. **Cloudflare.** The Cloudflare MCP server works from a Claude Code session, but the automation needs a scoped API token of its own, for the Terraform record and for cert-manager. Which tool manages the DNS record, and where the token comes from, is not decided. This draft assumes the Terraform Cloudflare provider with a token from the environment.
3. **AWS credentials.** Short-lived credentials, method not decided. The tool only requires that `aws sts get-caller-identity` works.
4. **Hostname and zone.** Not yet chosen.
5. **`PODIUM_TRUST_PROXY` value.** The number of hops (Cloudflare, NLB, ingress) is confirmed in P5.3 by reading the source address in Podium's Admin log. It is not assumed.
6. **Kubernetes version and node size.** To be set at P2.2 from current availability and cost.
7. **Argo CD revision.** `main` only moves on a release, so a build before the first release has nothing to sync. The default may need to be `development` until then.
