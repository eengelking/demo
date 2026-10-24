# SPEC: EKS platform bootstrap for Podium

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-24 | First draft. |
| 0.2 | 2026-10-24 | Cloudflare DNS moves out of Terraform into an agent and a skill. Adds the reporter port (4318) and the DNS-only hostname, the DNS-01 secret handling, agent-safe Terraform execution, the image build task, the Kubernetes MCP task, a single node on Kubernetes 1.37. Still provisional until the open questions in section 9 are answered. |

This project stands up a hardened Kubernetes platform on AWS from nothing and deploys [Podium](https://github.com/eengelking/podium) onto it. Terraform creates the network and an EKS cluster. Ansible hardens the nodes and keeps them hardened as they are replaced. Argo CD is bootstrapped and deploys Podium from a Helm chart in this repo. cert-manager issues a Let's Encrypt certificate for `demo.edengelking.com`.

It also shows that agents and skills can do specific work inside guardrails. A DNS agent (the what and why) and a DNS skill (the how) own the Cloudflare record through an MCP server, and a Kubernetes MCP server lets an agent check cert-manager before and after a certificate is requested. Claude Code runs Terraform without being able to read a secret.

One person runs it from a workstation with short-lived AWS credentials. The repo is public, and its history is meant to be read, so every stage is a separate, named, testable unit.

## 1. Goals and non-goals

### Goals

- One documented sequence takes an empty AWS region to Podium answering over HTTPS on `https://demo.edengelking.com`, with the reporter port reachable on `demo.edengelking.com:4318`.
- Every stage (state, network, cluster, add-ons, hardening, Podium) can be run, verified and torn down on its own.
- Nodes are hardened after they join, and a replaced node is re-hardened without anyone running anything.
- The Let's Encrypt environment (`staging` or `prod`) is one variable.
- The public hostname is one variable. The Ingress, the certificate and Podium's host settings are all derived from it.
- The DNS record is created, changed and removed only by the DNS agent, using the DNS skill, and only for that one hostname.
- Claude Code can plan and apply Terraform through approved commands without any credential, state value or token reaching its context.
- Teardown removes everything the tool created and is tested before anything else depends on it.
- No secret is ever in the repo, in Argo CD's UI or in a log.

### Non-goals (do NOT build these)

- High availability for Podium. It runs one replica on a SQLite volume. Other databases and multiple replicas are Podium's roadmap, not this tool's.
- Multi-region, multi-cluster or multi-account support.
- Managing Cloudflare with Terraform. The record belongs to the DNS agent.
- Using Cloudflare's proxy. The hostname is DNS-only (grey cloud), because the reporter port needs mutual TLS to reach the pod untouched.
- A pre-baked hardened node image (Packer). It is the better production pattern and is documented in `docs/DECISIONS.md`, not built.
- Bottlerocket or any node OS other than Amazon Linux 2023.
- A compliance score. The tool names the benchmark it applied and the controls it applied. It does not report a pass rate.
- Auto-approving Terraform against a real account. Every apply and destroy is a command a person approves.
- Reporting real Claude Code sessions to the deployed Podium. Only the Podium simulator reports to it.
- Authentication in front of the Podium console. Reads are open by design on a trusted network. An access layer is deferred (section 8).
- Cloudflare settings beyond the one record.
- Building the Podium image from anything but the Podium source, for `linux/amd64`, once per release of the tool.

## 2. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Infrastructure | Terraform 1.17 | State, plan review and drift detection. Separate roots per concern keep a destroy of one from touching the others. |
| State | S3 bucket with native locking (`use_lockfile`) | No DynamoDB table to create and tear down. |
| Region and arch | `us-west-2`, amd64 | Chosen by the maintainer. |
| Kubernetes | EKS 1.37 if available, one managed node | Chosen by the maintainer. The version is checked at P2.2. |
| Node OS | Amazon Linux 2023 EKS-optimized AMI | SSM agent is present, and a CIS benchmark exists for it. |
| Hardening | Ansible, applied through an SSM association | Re-applies to nodes the group replaces. A one-time run would be lost on the next recycle. |
| Storage | EBS CSI driver add-on, `gp3` StorageClass | Podium's SQLite volume needs a PersistentVolumeClaim. |
| GitOps | Argo CD, app-of-apps, tracking `development` until the first release | One bootstrap app owns the rest. `main` has nothing to sync before a release. |
| Ingress | ingress-nginx behind an NLB | Terminates TLS on 443 for the console. Buffering and timeouts are configurable for Podium's server-sent events. |
| Reporter port | NLB TCP listener on 4318, passed straight to the pod | Podium's reporter port uses its own CA and client certificates. Anything that terminates TLS would break it. |
| Certificates | cert-manager, DNS-01 solver, one `staging`/`prod` variable | Chosen by the maintainer. The solver needs a scoped Cloudflare API token in a cluster Secret. |
| DNS | Cloudflare, DNS-only record, managed by an agent and a skill through the Cloudflare MCP server | Shows an agent with a narrow job and guardrails. |
| Cluster introspection | A Kubernetes MCP server, read-only | Lets an agent check cert-manager resources without a shell on the cluster. |
| App packaging | A small Helm chart in this repo | Argo CD renders it. Values carry the hostname, image and settings. |
| Image registry | GitHub Container Registry, public package | No pull secret needed. |
| Scripting | Bash, `make` | One script or target per stage. |

## 3. Glossary

- **Stage:** one independently runnable unit of the build: state, network, cluster, add-ons, hardening, Podium.
- **Root:** a Terraform root module with its own state, one per stage that Terraform owns.
- **Issuer variable:** `letsencrypt_env`, either `staging` or `prod`.
- **Hostname variable:** `podium_hostname`, set to `demo.edengelking.com`.
- **Reporter port:** port 4318, Podium's optional TLS listener that accepts only reporters with a client certificate.
- **DNS agent / DNS skill:** the agent that decides what the record should be and why, and the skill that carries out the change.

## 4. Architecture

### Layout

```
terraform/
  state/        S3 bucket for remote state (local state, created once)
  network/      VPC, subnets, NAT
  cluster/      EKS, node group, IAM, EBS CSI add-on, SSM association
  addons/       Argo CD, ingress-nginx, cert-manager, issuers, Podium secret
ansible/
  roles/        node hardening role(s)
  playbooks/    the playbook the SSM association runs
gitops/
  apps/         Argo CD Application manifests (app-of-apps)
  charts/podium/ Helm chart for Podium
.claude/
  agents/       the DNS agent
  skills/       the DNS skill
scripts/        one script per stage, plus verify, teardown, token and secrets helpers
```

`network`, `cluster` and `addons` read each other's outputs through remote state. A stage never reads another stage's files.

### Traffic

```
browser   -- 443  --> NLB --> ingress-nginx (Let's Encrypt cert) --> Podium :4317 (plain HTTP)
reporter  -- 4318 --> NLB (TCP passthrough) -------------------------> Podium :4318 (mutual TLS)
```

Both go to `demo.edengelking.com`, a DNS-only record that points at the NLB. Podium's own CA and serving certificate live on its `/data` volume and are never exposed outside the pod except as the CA fingerprint in Admin. The serving certificate must include the hostname, so `PODIUM_ALLOWED_HOSTS` is set before Podium's first start.

### Hardening

An SSM association targets instances tagged for hardening and runs the playbook on a schedule and when an instance first becomes managed. The playbook is uploaded to S3 by Terraform. The role applies selected Level 1 server controls from the CIS Amazon Linux 2023 Benchmark and lists each applied control by ID. Controls that would break Kubernetes (for example disabling IP forwarding) are skipped and listed as deliberate deviations in `docs/DECISIONS.md`.

### Podium

Podium runs as a single-replica Deployment with a PersistentVolumeClaim mounted at `/data`. Settings come from the chart values:

- `PODIUM_ALLOWED_HOSTS` is the hostname variable.
- `PODIUM_TLS` is on, so the reporter port exists.
- `PODIUM_TRUST_PROXY` is the number of proxy hops in front of the pod, and only a proxy that overwrites `X-Forwarded-For` counts as a hop. With no Cloudflare proxy, the hop is ingress-nginx for the console. P5.3 confirms the value.
- `PODIUM_TOKEN` comes from a Kubernetes Secret that Terraform creates from a generated random value.

The ingress sets proxy buffering off and long read and send timeouts, because the browser view streams over server-sent events.

### Secrets

| Secret | Created by | Where it lives | Seen by Claude |
|---|---|---|---|
| Podium token | Terraform (random value) | Kubernetes Secret, and Terraform state | No |
| Cloudflare token for cert-manager | `scripts/set-cloudflare-token.sh`, which reads it from a hidden prompt the user types into | Kubernetes Secret only | No |
| AWS credentials | The user, with a named profile and short-lived credentials | `~/.aws`, never in a variable | No |

The Podium token is in Terraform state, so the state bucket is encrypted, versioned and not public. This is an accepted risk, recorded in `docs/DECISIONS.md`. `scripts/get-podium-token.sh` reads the Secret with `kubectl` and writes `.secrets/podium-token` with mode `0600`.

### Agent-safe execution

Claude Code runs Terraform and cluster commands only through `make` targets. Project permission rules allow those targets and deny reading `~/.aws`, `*.tfstate*`, real `*.tfvars`, `.secrets/` and `.env*`, and deny the commands that print state or secrets (`terraform show`, `terraform state`, `terraform output`, `kubectl get secret`). Targets log to files, and the logs are not read for secrets because the targets never print them. The deny rules are part of the repo, so they are reviewable.

### The DNS agent and skill

The agent decides whether a DNS change is needed and why, checks that the load balancer address exists, and asks the skill to make the change. The skill performs the change through the Cloudflare MCP server and then confirms it with a DNS lookup. Guardrails, enforced by the agent's tool list and the skill's steps:

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
| `kubernetes_version` | Defaults to `1.37` if available. |
| `node_instance_type` | One amd64 node large enough for Argo CD, ingress-nginx, cert-manager and Podium. |
| `gitops_revision` | The branch Argo CD tracks. Defaults to `development`. |
| `budget_limit_usd`, `budget_email` | The budget alarm. |

## 6. Feature flags

None.

## 7. Phased build plan

Phases run in order. Each ends in a state someone can observe. Tasks inside a phase that have no shared dependency can be worked in parallel by separate agents, each on its own issue branch.

### Phase 1 — Foundations

**Goal:** the repo can validate itself, runs Terraform safely, has an image to deploy and has remote state to build on.

#### P1.1 Repo skeleton and tooling checks
- **Scope:** top-level `Makefile`, `scripts/preflight.sh`, `terraform.tfvars.example` files, `.secrets/` gitignore entry. Nothing under `terraform/` resources.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `scripts/preflight.sh` exits non-zero and names the missing tool when `terraform`, `ansible`, `aws`, `kubectl` or `helm` is absent.
  2. `scripts/preflight.sh` exits non-zero when `aws sts get-caller-identity` fails for the configured profile.
  3. `make help` lists one target per stage.
  4. `git check-ignore .secrets/podium-token` succeeds.

#### P1.2 Agent-safe execution
- **Scope:** `.claude/settings.json` permission rules, and the `make` targets that wrap Terraform and cluster commands. Nothing under `terraform/` resources.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. A request from Claude to read `~/.aws/credentials`, a `*.tfstate` file, a real `*.tfvars`, `.secrets/` or `.env` is refused by a permission rule.
  2. Requests for `terraform show`, `terraform state`, `terraform output` and `kubectl get secret` are refused.
  3. `make plan-<stage>` runs a Terraform plan and writes its output to `logs/`, and the output contains no value marked sensitive.
  4. `make apply-<stage>` and `make destroy-<stage>` are the only apply and destroy paths, and each still requires a person to approve the command.
  5. The rules are listed in `docs/DECISIONS.md` with the reason for each.

#### P1.3 Remote state bucket
- **Scope:** `terraform/state/` only.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. `terraform apply` in `terraform/state/` creates a private, versioned, encrypted bucket with public access blocked.
  2. The other roots can configure it as a backend with `use_lockfile = true`, and a second concurrent `plan` fails with a lock error.
  3. `terraform destroy` in `terraform/state/` removes the bucket, including versions, when run through the teardown script.

#### P1.4 Podium image build and publish
- **Scope:** `scripts/build-image.sh` and the image reference in the example tfvars. Nothing in the Podium source repo is changed.
- **Depends on (in-phase):** P1.1
- **Acceptance:**
  1. The script builds the Podium image for `linux/amd64` and pushes it to GHCR as a public package, tagged with the Podium version.
  2. `podman manifest inspect` or `docker buildx imagetools inspect` on the pushed tag shows `linux/amd64`.
  3. A pull of the image works without logging in.
  4. The script prints no token, and uses an existing `gh` login.

**Exit criterion:** `make preflight` passes, a refused-read test for each denied path passes, the state bucket holds a lock file during a plan, and the image can be pulled anonymously.

### Phase 2 — Network and cluster

**Goal:** an empty, working EKS cluster.
**Needs from Phase 1:** the state bucket (P1.3), the safe `make` targets (P1.2) and the image tag (P1.4).

#### P2.1 Network
- **Scope:** `terraform/network/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `terraform plan` shows a VPC with public and private subnets across at least two availability zones, tagged for EKS and load balancers.
  2. Outputs expose VPC ID and subnet IDs, and the cluster root reads them only through remote state.
  3. `terraform destroy` leaves no VPC, subnet, NAT gateway or elastic IP tagged with `name`.

#### P2.2 Cluster and node
- **Scope:** `terraform/cluster/` EKS, node group, IAM roles. Not the add-ons and not the SSM association.
- **Depends on (in-phase):** P2.1
- **Acceptance:**
  1. The cluster runs the version in `kubernetes_version`. If EKS does not offer 1.37 yet, the plan fails with a message naming the versions it does offer.
  2. A managed node group with one Amazon Linux 2023 amd64 node joins, and `kubectl get nodes` shows it `Ready`.
  3. The node carries the tag the hardening association targets, and has the SSM managed-instance policy attached.
  4. `aws eks update-kubeconfig` produces a working context, and the script that does it does not print credentials.
  5. The instance type is a variable, and the one chosen leaves memory free after Argo CD, ingress-nginx, cert-manager and Podium are running.

#### P2.3 EBS CSI driver and StorageClass
- **Scope:** the EBS CSI add-on and its IAM role (Pod Identity or IRSA) in `terraform/cluster/`, plus the `gp3` StorageClass.
- **Depends on (in-phase):** P2.2
- **Acceptance:**
  1. A test PVC against the `gp3` StorageClass binds, and a pod writes and reads a file on it.
  2. The CSI controller uses a dedicated role with only the volume permissions it needs.

**Exit criterion:** `kubectl get nodes` shows a `Ready` node and a throwaway PVC binds and is writable.

### Phase 3 — Node hardening

**Goal:** the node is hardened, and a replacement is hardened automatically.
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
  2. After apply, the node's association status is `Success`.
  3. Terminating the node results in a replacement that reports `Success` without anyone running a command. With one node this is a short outage of everything on the cluster, and the test is run only before Podium is deployed.

#### P3.3 Hardening verification script
- **Scope:** `scripts/verify-hardening.sh` only.
- **Depends on (in-phase):** P3.2
- **Acceptance:**
  1. The script checks each applied control on the node and prints a list of control IDs with pass or fail. It prints no percentage.
  2. It exits non-zero if any applied control fails.

**Exit criterion:** `scripts/verify-hardening.sh` passes, then passes again after the node is terminated and replaced.

### Phase 4 — GitOps, add-ons and cluster visibility

**Goal:** Argo CD, ingress, certificates and a read-only view of the cluster work, with nothing deployed on top yet.
**Needs from Phase 2:** a working cluster and the `gp3` StorageClass (P2.3).

#### P4.1 Argo CD bootstrap
- **Scope:** `terraform/addons/` Argo CD release and the root app-of-apps Application. `gitops/apps/` root only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Argo CD is running, and the root Application syncs from this repo at `gitops_revision`.
  2. The Argo CD admin secret is not printed by any script and is not in Git.
  3. Sync policy is automated with prune and self-heal, and a manual change to a managed resource is reverted.

#### P4.2 Ingress controller and load balancer
- **Scope:** an Argo CD Application for ingress-nginx under `gitops/apps/`, including the NLB listeners.
- **Depends on (in-phase):** P4.1
- **Acceptance:**
  1. An NLB is provisioned with listeners on 80, 443 and 4318, and the controller reports ready.
  2. The 4318 listener passes TCP straight through to Podium's pod and does not go through the ingress controller.
  3. The controller configuration sets proxy buffering off and read and send timeouts of at least one hour for the Podium ingress.

#### P4.3 cert-manager and issuers
- **Scope:** an Argo CD Application for cert-manager, the `ClusterIssuer` for staging and prod under `gitops/`, and `scripts/set-cloudflare-token.sh`.
- **Depends on (in-phase):** P4.1
- **Acceptance:**
  1. Both issuers exist, and the active one is chosen by `letsencrypt_env` alone.
  2. `scripts/set-cloudflare-token.sh` reads the token from a hidden prompt, creates the Secret in the cert-manager namespace, and prints nothing from it. The token is never an argument, an environment variable or a file.
  3. The token does not appear in Argo CD, in Git or in any log.
  4. A test `Certificate` against the staging issuer reaches `Ready` for a name in the zone.

#### P4.4 Kubernetes MCP server
- **Scope:** the MCP server configuration in `.mcp.json` and the read-only service account it uses. Nothing else in the cluster.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The server can list and describe `Certificate`, `CertificateRequest`, `Order` and `Challenge` resources, and pods and events in the add-on namespaces.
  2. A request to read a Secret, or to create, change or delete anything, is refused.
  3. An agent can use it to report why a certificate is not `Ready`.

**Exit criterion:** a test `Certificate` for a name in the zone becomes `Ready` through the staging issuer, the NLB has an address, and the Kubernetes MCP server reports the certificate's state.

### Phase 5 — Podium and DNS

**Goal:** Podium answers on the public hostname, and reporters can connect to the reporter port.
**Needs from Phase 4:** Argo CD (P4.1), the load balancer (P4.2), the issuers (P4.3) and the Kubernetes MCP server (P4.4).

#### P5.1 Podium Helm chart
- **Scope:** `gitops/charts/podium/` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. `helm lint` and `helm template` pass with the example values.
  2. The chart renders one replica, a PVC mounted at `/data`, a Service on 4317 and 4318, an Ingress for the hostname and a Certificate for it.
  3. `PODIUM_ALLOWED_HOSTS`, `PODIUM_TLS` and `PODIUM_TRUST_PROXY` come from values. `PODIUM_TOKEN` comes from a Secret reference and never from values.
  4. The image reference has no default, and rendering without it fails with a named error.

#### P5.2 Token generation and retrieval
- **Scope:** the generated token and Secret in `terraform/addons/`, and `scripts/get-podium-token.sh`.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. Terraform creates the Secret in the Podium namespace from a generated random value, and the value is not in any file in the repo.
  2. `scripts/get-podium-token.sh` writes `.secrets/podium-token` with mode `0600` and prints no token to the terminal.
  3. The token is not visible in the Argo CD UI.

#### P5.3 Podium application
- **Scope:** the Argo CD Application that points at the chart.
- **Depends on (in-phase):** P5.1, P5.2
- **Acceptance:**
  1. One variable, `podium_hostname`, sets the Ingress host, the certificate name and `PODIUM_ALLOWED_HOSTS`.
  2. Podium reaches `Ready`, and the PVC is bound.
  3. The Admin log shows the real client address as the source for a console request, and `PODIUM_TRUST_PROXY` is set to the value that makes that true.

#### P5.4 DNS agent and skill
- **Scope:** `.claude/agents/` and `.claude/skills/` for the DNS work. No Terraform, no cluster resources.
- **Depends on (in-phase):** P4.4
- **Acceptance:**
  1. The agent's tool list contains only the Cloudflare MCP tools, the Kubernetes MCP tools and read-only shell lookups. It has no Terraform and no file-writing tools.
  2. Asked to point `demo.edengelking.com` at the NLB, it reads the NLB address from the cluster, makes the change through the skill, and confirms it with a DNS lookup.
  3. Asked to change any other record, zone or setting, it refuses and says why.
  4. Asked to use an address typed into the prompt, it refuses and uses the cluster's.
  5. The record it creates is DNS-only.
  6. Running it again when the record is already correct makes no change and says so.

#### P5.5 Certificate and reporter port
- **Scope:** the production certificate, the reporter port, and the checks for both. No new components.
- **Depends on (in-phase):** P5.3, P5.4
- **Acceptance:**
  1. With the staging issuer, `curl` to `https://demo.edengelking.com/` returns Podium's page with the staging certificate.
  2. With the prod issuer, the browser shows a trusted certificate. The agent uses the Kubernetes MCP server to confirm the `Certificate` is `Ready` before and after the switch.
  3. A reporter config created in Admin > People from the public address connects over 4318, and a Podium simulator session appears on the wall.
  4. A connection to 4318 without a client certificate is refused.

**Exit criterion:** `https://demo.edengelking.com/` serves Podium with a valid certificate, and the Podium simulator can report a session to it over the reporter port.

### Phase 6 — Operability

**Goal:** the whole thing can be run, checked and removed safely.
**Needs from Phase 5:** a working deployment to verify and tear down.

#### P6.1 End-to-end verify script
- **Scope:** `scripts/verify.sh` only.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. It checks the node, Argo CD sync status, the certificate, HTTPS, the reporter port and the Podium health response, and prints one line per check.
  2. It holds a server-sent events connection open for ten minutes and fails if it drops.
  3. It exits non-zero if any check fails.

#### P6.2 Teardown
- **Scope:** `scripts/teardown.sh`, and the order in which roots are destroyed.
- **Depends on (in-phase):** none — ready to start
- **Acceptance:**
  1. The script asks for the name of the environment before destroying and does nothing on a mismatch.
  2. It stops with a message if `demo.edengelking.com` still resolves to the NLB, telling the user to ask the DNS agent to remove the record first.
  3. It destroys add-ons, cluster, network and state in that order, so no load balancer is orphaned.
  4. After a full run, `aws` queries by tag find no VPC, cluster, load balancer, volume, elastic IP or bucket for that name.
  5. A second run on an already-clean account exits zero.

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
  2. `docs/DECISIONS.md` has an entry for each decision named in this spec, including the DNS-only hostname and the DNS-01 choice.
  3. `docs/ARCHITECTURE.md` matches what was built, and says where it departs from this spec.

**Exit criterion:** from an empty region, the documented sequence reaches a passing `scripts/verify.sh`, and `scripts/teardown.sh` returns the region to empty.

## 8. Explicitly out of scope / deferred

- An access layer in front of the Podium console. The design must not preclude one, since the ingress is the single entry point for the console.
- Moving Podium to another database and more than one replica, when Podium supports it.
- A baked hardened image in place of the SSM association. The role is written so it can be reused in an image build.
- Policy checks on `terraform plan` in CI (security, cost, tagging).
- Cloudflare's proxy in front of the console. Possible later with a second hostname for the reporter port.

## 9. Open questions

1. **AWS credentials.** Short-lived credentials, method not decided. The tool only requires that `aws sts get-caller-identity` works for the named profile.
2. **Kubernetes 1.37.** Whether EKS offers it cannot be checked until the AWS CLI is configured. P2.2 fails with a clear message if it does not, and the version is a variable.
3. **Node size.** One node has to carry Argo CD, ingress-nginx, cert-manager and Podium. The instance type is chosen at P2.2.
4. **Which Kubernetes MCP server.** Not chosen. P4.4 picks one that supports read-only access and records the choice in `docs/DECISIONS.md`.
5. **Podium TLS on Kubernetes.** Podium's docs describe where the CA lives for `podiumctl`'s container mode and for standalone mode. Where it lives in a pod, and that it survives a restart on the volume, is confirmed in P5.5.
6. **Cloudflare token scope.** The cert-manager token needs DNS edit access to one zone. Creating it is a step for the user, outside this repo.
