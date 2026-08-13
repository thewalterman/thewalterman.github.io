---
title: "A Sandboxed AI Ops Bot for the Homelab"
date: 2026-08-13
draft: false
tags: ["ansible", "k3s", "kubernetes", "rbac", "podman", "security", "ai", "llm", "homelab"]
description: "Installing Hermes Agent — a self-hosted, chat app reachable AI ops assistant — on the homelab node via Ansible, then taking away almost everything it could do wrong: a rootless Podman sandbox for command execution and a dedicated Kubernetes identity."
ShowToc: true
---

The [homelab](/posts/homelab/) runs on a single k3s node, provisioned and patched entirely through [OpenTofu and Ansible](/posts/homelab-infra/). The one piece that didn't fit that pattern was giving an AI agent enough access to actually help operate it — `kubectl`, `helm`, `flux` — without also giving it the run of the box. This is that piece: [Hermes Agent](https://github.com/NousResearch/hermes-agent), installed by a Ansible playbook alongside the OS patching and k3s bootstrap ones, reachable over chat app, and boxed in tightly enough that a bad command from it can't reach past the cluster it's meant to manage.

---

## Another Playbook, Same Shape as the Others

`bootstrap-hermes.yml` follows the same pattern as `bootstrap-k3s.yml`: one more thing that lives at the OS level, outside OpenTofu's resource graph, installed and kept in sync by Ansible.

```bash
ansible-playbook -i ansible/inventory.yml ansible/bootstrap-hermes.yml
```

It runs Hermes's own installer, pushes a handful of versioned files onto the node, and manages a `systemd --user` service with linger enabled — so the gateway survives a reboot without anyone ever logging in interactively:

```yaml
- name: check for existing hermes install
  stat:
    path: "{{ hermes_home }}/hermes-agent/venv/bin/python"
  register: hermes_venv

- name: run official hermes installer
  shell: curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
  when: not hermes_venv.stat.exists

- name: template SOUL.md, AGENTS.md, config.yaml
  copy:
    src: "hermes/files/{{ item }}"
    dest: "{{ hermes_home }}/{{ item }}"
  loop: [SOUL.md, AGENTS.md, config.yaml]

- name: enable linger so the user service survives reboot without login
  become: true
  command: "loginctl enable-linger {{ ansible_user }}"
```

Hermes's own docs draw a deliberate line between two files, and the split is worth keeping: **`SOUL.md`** is identity and tone — who the agent is, how it talks — and is meant to stay durable across projects. **`AGENTS.md`** is operational rules specific to this homelab — how to use `kubectl`, when to ask before acting, GitOps discipline. Mixing them means every future tweak to "how should it talk" risks dragging homelab-specific instructions along with it, and vice versa.

---

## Guardrails Before Sandboxing

Two config layers went in before anything about isolation:

```yaml
approvals:
  mode: manual
  deny:
    - "rm -rf*"
    - "*k3s-killall.sh*"
    - "*flux uninstall*"
    - "*ufw*"
    - "reboot*"
    - "shutdown*"
```

`mode: manual` means the agent asks before running anything it judges risky; the `deny` list is a hard floor underneath that judgment, patterns that never execute regardless of what the model decides. Cheap to write, and worth having — but it's a list of *known* bad commands. It says nothing about a command nobody thought to list.

---

## The Real Question: What Can It Actually Reach

By default Hermes can execute commands directly on the node. That's the same privilege level as an interactive SSH session — a prompt injection from a page it fetched, or the model simply doing the wrong thing with real conviction, lands on the same host that runs the entire cluster. The `deny` list above helps against commands *someone thought to write down*; it does nothing about the shape of the access itself.

The fix has two independent layers: sandbox where commands execute, and scope what they can do once they're in there.

---

## Layer 1: Rootless Podman Instead of the Bare Host

Hermes's `terminal.backend` config can route command execution through a container instead of running locally. The obvious choice is Docker — except Docker means a daemon running as root, permanently, and giving the agent's own unprivileged user access to it means adding that user to the `docker` group, which is well-known shorthand for *give this user root*. That's a strange trade for a tool whose whole job is running as an unprivileged `systemd --user` service.

[Podman](https://podman.io/) fixes the shape of the problem rather than working around it: **no daemon**, and **rootless by default** — containers run inside the calling user's own namespace, no group membership that's secretly root-equivalent.

```yaml
terminal:
  backend: docker    # the config key didn't change
  docker_image: "localhost/hermes-sandbox:latest"
```

```ini
# hermes-gateway.service
Environment="HERMES_DOCKER_BINARY=podman"
```

Installing it is one Ansible task ahead of everything else in the playbook:

```yaml
- name: ensure podman is installed (rootless sandbox runtime for Hermes terminal.backend)
  become: true
  apt:
    name: podman
    state: present

- name: verify rootless podman works for this user
  command: podman info
  changed_when: false
```

---

## Building the Sandbox Image

The sandbox needs `kubectl`, `helm`, and `flux` — none of which exist in a generic base image. The first attempt at reusing what was already on the node hit an odd wall: `kubectl` on the host turned out to be a symlink to the `k3s` binary itself (`k3s kubectl ...`), not a standalone executable, so bind-mounting it into an unrelated container wasn't really an option. Standalone binaries, pinned to the versions already running on the node, went into a small image instead:

```dockerfile
FROM debian:bookworm-slim

ARG KUBECTL_VERSION=v1.34.6
ARG HELM_VERSION=v4.1.3
ARG FLUX_VERSION=v2.9.4
ARG TARGETARCH=arm64

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl tar \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/${TARGETARCH}/kubectl" -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl

RUN curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-${TARGETARCH}.tar.gz" | tar -xz -C /tmp \
    && mv "/tmp/linux-${TARGETARCH}/helm" /usr/local/bin/helm

RUN curl -fsSL "https://github.com/fluxcd/flux2/releases/download/${FLUX_VERSION}/flux_${FLUX_VERSION#v}_linux_${TARGETARCH}.tar.gz" \
    | tar -xz -C /usr/local/bin flux
```

`TARGETARCH=arm64` because the node is an Oracle Ampere instance — easy to forget when every example on the internet defaults to `amd64`. The Ansible task rebuilds it only when the Dockerfile changes or the tag is missing:

```yaml
- name: check whether the hermes-sandbox image already exists
  command: podman image exists localhost/hermes-sandbox:latest
  register: hermes_sandbox_image
  failed_when: false
  changed_when: false

- name: build the hermes-sandbox image (Dockerfile changed, or image/tag missing)
  command: podman build -t localhost/hermes-sandbox:latest "{{ hermes_home }}/sandbox"
  when: hermes_sandbox_context.changed or hermes_sandbox_image.rc != 0
```

---

## The Networking Trap

k3s's own kubeconfig points at `https://127.0.0.1:6443`, which is correct on the node and wrong the instant it's read from inside a container with its own network namespace — `127.0.0.1` in there is the *container's* loopback, nothing is listening on it. The kubeconfig generated for the sandbox rewrites the server address to the node's LAN IP instead of the loopback:

```yaml
- name: generate a sandbox kubeconfig from the dedicated hermes ServiceAccount
  become: true
  shell: |
    CA=$(k3s kubectl get secret hermes-token -n hermes -o jsonpath='{.data.ca\.crt}')
    TOKEN=$(k3s kubectl get secret hermes-token -n hermes -o jsonpath='{.data.token}' | base64 -d)
    cat > "{{ hermes_home }}/kubeconfig-sandbox" <<EOF
    apiVersion: v1
    kind: Config
    clusters:
      - name: homelab
        cluster:
          server: https://{{ ansible_facts.default_ipv4.address }}:6443
          certificate-authority-data: ${CA}
    users:
      - name: hermes
        user:
          token: ${TOKEN}
    contexts:
      - name: hermes
        context: {cluster: homelab, user: hermes}
    current-context: hermes
    EOF
```

---

## Layer 2: Cluster Admin, Not Node Admin

Sandboxing execution isn't the whole story. Mounting k3s's real admin kubeconfig into that container, read-only, makes the isolation purely cosmetic — a compromised command inside the sandbox can still `kubectl create` a pod with `hostPath: /` or `privileged: true` and walk straight back onto the node it was supposedly kept off of. RBAC and Podman are answering two different questions: Podman contains a *command*, RBAC has to contain what that command's *credentials* can do.

In the GitOps repo we have Pod Security `baseline` on every namespace and this already blocks `privileged: true`, `hostPath` volumes, and host namespaces (`hostNetwork`/`hostPID`/`hostIPC`) at admission, before RBAC even enters the picture. That closes the most severe escape route regardless of what the mounted kubeconfig can do. What was still missing was an identity that isn't cluster-admin in the first place.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hermes
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: hermes
  namespace: hermes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: hermes-edit
subjects:
  - kind: ServiceAccount
    name: hermes
    namespace: hermes
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
metadata:
  name: hermes-token
  namespace: hermes
  annotations:
    kubernetes.io/service-account.name: hermes
type: kubernetes.io/service-account-token
```

---

## Applying RBAC From Ansible, Not GitOps

The cluster's applications are managed by Flux, and the instinct was to put this manifest there too, next to Keycloak and everything else. It doesn't fit: Flux isn't bootstrapped yet at the point `bootstrap-hermes.yml` runs — the order is Tofu → k3s → *then* Flux, manually, once — and Hermes itself already lives outside Flux's resource graph, same as k3s. Routing the RBAC identity through GitOps would mean Hermes's own bootstrap depends on a completely different repo having already been reconciled.

So the playbook applies it directly:

```yaml
- name: apply Hermes RBAC manifest
  become: true
  command: "k3s kubectl apply -f {{ hermes_home }}/hermes-rbac.yaml"
  register: hermes_rbac_apply
  changed_when: "'unchanged' not in hermes_rbac_apply.stdout"

- name: wait for the hermes ServiceAccount token to be populated
  become: true
  command: "k3s kubectl get secret hermes-token -n hermes -o jsonpath={.data.token}"
  register: hermes_token_check
  until: hermes_token_check.stdout | length > 0
  retries: 10
  delay: 2
```

---

## What's Actually Contained Now

| Question | Before | After |
| --- | --- | --- |
| Where do commands run? | Directly on the host | Inside a rootless Podman container |
| What can a compromised command reach on the node? | Everything the user can | Nothing — no host filesystem, no systemd, no other processes |
| What Kubernetes identity does it carry? | The cluster's own admin kubeconfig | A dedicated `hermes` ServiceAccount |
| Can it read/restart/patch workloads? | Yes | Yes, across every namespace |
| Can it list nodes, or touch RBAC objects? | Yes | No |
| Can it create a pod that reaches the host directly? | Yes | No — blocked at admission by Pod Security `baseline`, independent of RBAC |

## Takeaways

- **Sandboxing execution and scoping credentials are two separate problems.** A container with an admin kubeconfig mounted into it is not contained — it's the same power with extra steps.
- **Admission control (Pod Security `baseline`) closes the escape hatch RBAC scoping alone can't** — the difference between "cluster admin" and "root on the node" is enforced there, not in the RBAC verbs.
- **Not everything belongs in GitOps.** A resource needed to bootstrap a component that itself sits outside the GitOps graph belongs with that component's own bootstrap tooling, not wedged into a repo that can't run yet.
