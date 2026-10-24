# demo

Stands up a hardened EKS cluster on AWS and deploys [Podium](https://github.com/eengelking/podium) onto it through GitOps.

- Terraform builds the network and the cluster in `us-west-2`.
- Ansible hardens the nodes.
- Argo CD is bootstrapped and deploys Podium.
- Cloudflare DNS is managed as code, and cert-manager issues a Let's Encrypt certificate.

The project is being built from a spec. See [`PROJECT.md`](PROJECT.md) for what it is and the rules it follows, and [`docs/`](docs/) for the spec, architecture and decisions as they are written.
