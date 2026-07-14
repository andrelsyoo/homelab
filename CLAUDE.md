# CLAUDE.md

## What this repo is

Public portfolio version of a private homelab GitOps repo (`london`). This repo is
**documentation only** — a single `README.md` describing the architecture, stack, and
notable incidents. There is no code to build, test, or run here.

The private `london` repo holds the actual Terraform/Ansible/GitOps source and detailed
internal runbooks (credentials, internal IPs, hostnames, node-by-node debugging steps).
When asked to reflect a change or incident from `london` here, pull the *narrative and
technical substance* over, not the raw internal details:

- Keep specific internal IPs, hostnames (`*.london.internal`), credentials, and Tailscale
  details out of this repo.
- Node names (`talos-n1a-187`, etc.) are fine — they already appear here and carry no
  sensitive information on their own.
- Write incidents as concise "war story" bullets under **Interesting challenges** — what
  broke, why, and the actual fix — matching the tone of the existing entries. This is a
  portfolio; the value is demonstrating real debugging depth, not just listing tools.

## Tone

The intro paragraph explicitly frames this repo as also serving as a portfolio piece —
keep that in mind when editing: technically accurate, but written for a reader evaluating
engineering judgment, not just an internal reference note to self.

## Conventions

- Commit style: `docs(readme): <description>` (this repo only ever touches `README.md`).
- Never invent architecture or numbers not confirmed from the actual `london` repo/cluster
  — this is a factual account of a real system.
