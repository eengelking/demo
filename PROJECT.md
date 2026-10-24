# Project

## Project Overview

This repo is a tool that stands up a hardened Kubernetes platform on AWS from nothing and deploys [Podium](https://github.com/eengelking/podium) onto it. Terraform creates the network and an EKS cluster in `us-west-2` on amd64 nodes. Ansible hardens the nodes. Argo CD is bootstrapped and deploys Podium through GitOps. Cloudflare DNS is managed as code, and cert-manager issues a Let's Encrypt certificate. It is for one person to run end to end, and for anyone to read: the whole history is meant to be reviewable.

It ships as this repo, run from a workstation with short-lived AWS credentials and a scoped Cloudflare token. Nothing is published to a registry. The Podium image comes from the maintainer's own build and is pulled from a registry the spec names. `docs/SPEC.md` is the source of truth for intended behavior; until it exists, ask before assuming.

## Do Not

- Write the name of any person or company other than the repo owner and Podium into any file, commit message, issue or PR in this repo. Tool names that the build needs (AWS, Terraform, Ansible, Argo CD, Cloudflare, Let's Encrypt) are fine. This applies to docs, comments and examples too.
- Commit state files, `*.tfvars` with real values, kubeconfigs, tokens, account IDs or any credential. This repo is public. See "Working Style" for how secrets are kept out.
- Point a real Claude Code reporter at the deployed Podium instance. Only the Podium simulator reports to it.
- Run `terraform apply`, `terraform destroy` or the teardown script against a real account without the user's explicit go-ahead in that moment. A prior approval does not carry forward.
- Claim a compliance score for the node hardening. Name the benchmark that was applied and stop there.

## Branching and Workflow Overrides

None. This project follows CLAUDE.md as written, using the development-branch workflow: PRs target `development`, and `main` moves only on a release PR.

## Skills

| Skill | Load it when... | Orchestrator | Sub-agent |
|---|---|---|---|
| `eengelking:spec-writing` | Drafting `docs/SPEC.md` or decomposing it into issues | Writes the spec and files the issues | N/A |
| `eengelking:documentation-voice-guide` | Writing any prose: README, docs, commit messages, PR and issue text | Yes | Yes |

## Working Style

- Every stage (network, cluster, add-ons, node hardening, GitOps bootstrap, DNS and certificates, teardown) has its own script or `make` target, so it can be run and tested alone.
- Terraform state lives in S3 with locking. State is split per concern: network, cluster, add-ons.
- The Let's Encrypt issuer is one variable, `staging` or `prod`. The public hostname is one variable, and both the Cloudflare record and Podium's `PODIUM_ALLOWED_HOSTS` and `PODIUM_TRUST_PROXY` settings derive from it.
- Secrets (the Podium admin token, the Cloudflare token) are generated or read from the environment at run time. They never appear in the repo, in Argo CD's UI or in logs.
- The teardown script is part of the deliverable and gets tested. A leftover cluster bills hourly.
- Build logs go to a file, so a terminal pane running a build is safe to show on a screen share.

## Standards

None yet. Decisions about node hardening, state layout and secrets handling are recorded in `docs/DECISIONS.md` as they are made.

## Releases

This project does not cut releases yet. If that changes, the `github-release` skill owns the steps and this section records what is specific.

## Delegation and Checks

None yet.
