---
name: deploy-on-cluster
description: Help deploy tools on the infodom-IaC cluster. Use this skill when the user wants to deploy a new Helm-based tool into gitops/sre-tools, scaffold the standard deployment files (deploy.sh, helm chart, Tailscale ingress), or understand how the gitops deployment setup works. Activate whenever the user mentions deploying an sre-tool, adding a tool to the cluster, exposing something via Tailscale, or asks how gitops/sre-tools is structured.
---

# deploy-on-cluster

Help the user deploy tools on the **infodom-IaC** Kubernetes cluster, following the
conventions used in `gitops/sre-tools/`. This skill either **scaffolds a new tool**
(generating the standard files) or **answers questions** about how the setup works.

**Input:** $ARGUMENTS

The infodom-IaC repository is at `/home/epiflight/Desktop/startup/infodom/infodom-IaC/`.
All sre-tools live under `gitops/sre-tools/`.

---

## Step 1 — Understand the existing setup

Before doing anything, read how the current tools are structured so your output matches
the house style. List the existing tools and inspect one or two as references:

```bash
cd /home/epiflight/Desktop/startup/infodom/infodom-IaC
ls gitops/sre-tools/
find gitops/sre-tools/prometheus gitops/sre-tools/wikijs -type f | sort
```

Read at least these reference files:

- `gitops/sre-tools/prometheus/deploy.sh` — the `deploy.sh` standard
- `gitops/sre-tools/wikijs/helm/templates/service-tailscale.yaml` — Tailscale exposure
- `gitops/sre-tools/prometheus/helm/templates/ingressroute.yaml` — Traefik exposure
- `gitops/sre-tools/wikijs/helm/Chart.yaml` and `values.yaml` — wrapper-chart pattern
- `gitops/sre-tools/wikijs/README.md` — README style

---

## Step 2 — Ask what the user needs

Use the **AskUserQuestion** tool (never plain text) to ask:

> Do you want to **deploy a new tool** on the cluster, or do you have **questions about
> how the deployment setup works**?

- **Questions only** → go to [Step 3](#step-3--answer-questions-mode).
- **Deploy a new tool** → go to [Step 4](#step-4--gather-requirements-deploy-mode).

---

## Step 3 — Answer questions (questions mode)

Use the reference files from Step 1 to explain how things work. Common topics:

- **Repository layout.** Each tool is a directory under `gitops/sre-tools/<tool>/` with a
  `deploy.sh` at its root and a `helm/` wrapper chart.
- **Wrapper-chart pattern.** `helm/Chart.yaml` declares the upstream chart as a
  `dependencies:` entry (optionally `alias:`-ed). Local `helm/templates/` add cluster-specific
  resources (Tailscale ingress, Traefik IngressRoute, RBAC, etc.). `helm/values.yaml` holds
  config; values for the subchart are nested under the dependency name/alias.
- **Secrets.** Encrypted with SOPS via the `helm-secrets` plugin, committed as
  `secrets-encrypted.yaml`. Plain secret files are never committed.
- **Exposure.** Internal-only tools use a Traefik `IngressRoute`; company-wide access uses a
  **Tailscale** ingress (`ingressClassName: tailscale`) reachable at
  `https://<hostname>.baiji-wall.ts.net`.
- **deploy.sh.** A thin wrapper that runs `helm dependency build/update` then
  `helm secrets upgrade ... --create-namespace -n "$NAMESPACE"`.

Answer the user's question grounded in the actual files. Do not invent behavior — read the
relevant file if you are unsure.

---

## Step 4 — Gather requirements (deploy mode)

Use the **AskUserQuestion** tool to collect everything needed. Ask in focused batches:

1. **Tool name** — the directory name under `gitops/sre-tools/` and the Helm release name
   (kebab-case, e.g. `uptime-kuma`). Often prefixed `locumo-` for the release; follow the
   user's preference.
2. **Helm chart source** — where the upstream chart comes from:
   - Helm repo: chart **name**, **version**, and **repository URL**
     (e.g. `name: uptime-kuma`, `version: 2.x`, `repository: https://helm.irsigler.cloud`), or
   - OCI registry reference, or
   - "no upstream chart — these are raw manifests I'll provide".
3. **Namespace** — the Kubernetes namespace (defaults to the tool name).
4. **Container image** (only if relevant / overriding the chart default) — repository + tag.
5. **Persistence** — does it need a PVC? If yes, size and storage class
   (the cluster uses `longhorn`).
6. **Exposure** — use AskUserQuestion with these options:
   - **Tailscale** — accessible to the rest of the company at
     `https://<hostname>.baiji-wall.ts.net`. Ask for the desired **hostname** and the
     **service port**.
   - **Traefik IngressRoute** — internal cluster routing. Ask for the **host** and
     **entryPoint**.
   - **None** — ClusterIP only, access via `kubectl port-forward`.
7. **Secrets** — does it need any secrets (DB password, API token, etc.)? If yes, note the
   keys; the user will encrypt them later with `helm secrets encrypt`.

Confirm the gathered summary back to the user before writing anything.

---

## Step 5 — Scaffold the tool directory

Create `gitops/sre-tools/<tool>/` with the files below. **Match the existing tools exactly.**

> Note: this skill **scaffolds files only** — it does not commit, open a PR, or run helm.
> Leave committing and deploying to the user.

### `deploy.sh` (required — follow the standard precisely)

```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE=<namespace>

helm dependency build helm/ || helm dependency update helm/
helm secrets upgrade <release-name> -f helm/values.yaml -f helm/secrets-encrypted.yaml helm/ --create-namespace -n "$NAMESPACE"
```

Rules for `deploy.sh`:
- First line is the shebang `#!/usr/bin/env bash`.
- Second line is `set -euo pipefail`.
- Define a `NAMESPACE=` variable (use `NAMESPACE_DEV` / `NAMESPACE_PROD` only if the tool
  deploys to multiple namespaces, like `alert-bot`).
- Always reference `"$NAMESPACE"` (quoted) in the helm command.
- Run `helm dependency build helm/ || helm dependency update helm/` before upgrading when the
  chart has dependencies. Omit this line if there is no upstream dependency.
- Use `helm secrets upgrade` (not `install`). Drop the `-f helm/secrets-encrypted.yaml`
  argument if the tool has no secrets.
- `chmod +x deploy.sh` after creating it.

### `helm/Chart.yaml`

```yaml
apiVersion: v2
name: <release-name>
description: A Helm chart for deploying <Tool>
type: application
version: 0.1.0
appVersion: "<app-version>"
dependencies:
- name: <upstream-chart>
  version: "<chart-version>"
  repository: "<repo-url>"
  alias: <alias>   # optional
```

Omit `dependencies:` entirely if the tool is raw manifests only.

### `helm/values.yaml`

Subchart config nested under the dependency name/alias; local template values at the top
level. Mirror the style of `wikijs/helm/values.yaml`. Example skeleton:

```yaml
<alias>:
  image:
    repository: <repo>
    tag: "<tag>"
  service:
    type: ClusterIP
    port: <port>
  persistence:        # only if needed
    enabled: true
    size: 5Gi
    storageClass: longhorn

service:
  name: <release-name>
  port: <port>

tailscale:            # only if exposing via Tailscale
  serviceName: <tool>-tailscale
  hostname: <hostname>
  tags: "tag:k8s"
  domain: "baiji-wall.ts.net"
```

### `helm/templates/` exposure resource

**If Tailscale** — create `helm/templates/service-tailscale.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Values.tailscale.serviceName }}
  annotations:
    tailscale.com/tags: "{{ .Values.tailscale.tags }}"
spec:
  ingressClassName: tailscale
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ .Values.service.name }}
            port:
              number: {{ .Values.service.port }}
  tls:
  - hosts:
    - {{ .Values.tailscale.hostname }}
```

**If Traefik** — create `helm/templates/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: {{ .Values.ingress.name }}
spec:
  entryPoints:
    - {{ .Values.ingress.entryPoint }}
  routes:
    - match: Host(`{{ .Values.ingress.host }}`)
      kind: Rule
      services:
        - name: {{ .Values.service.name }}
          port: {{ .Values.service.port }}
```

Add the matching `ingress:` / `service:` blocks to `values.yaml`.

### `helm/secrets-encrypted.yaml` (only if the tool has secrets)

Do **not** generate or commit plaintext secrets. Instead, write a placeholder
`secrets-plain.yaml` example and tell the user to encrypt it:

```bash
cd gitops/sre-tools/<tool>/helm
cp secrets-plain.example.yaml secrets-plain.yaml   # fill in real values
helm secrets encrypt secrets-plain.yaml > secrets-encrypted.yaml
rm secrets-plain.yaml
```

### `README.md`

Short README in the style of `wikijs/README.md`: what the tool is, prerequisites,
first-time install, upgrade, exposure URL, and the secrets table (if any).

---

## Step 6 — Verify and hand off

After scaffolding, sanity-check the chart renders:

```bash
cd /home/epiflight/Desktop/startup/infodom/infodom-IaC/gitops/sre-tools/<tool>
helm dependency build helm/ || helm dependency update helm/
helm template . -f helm/values.yaml | head -50
```

> The `helm secrets` / cluster steps require the `helm-secrets` plugin, a SOPS age key, and
> kubectl context. If those aren't available locally, skip the render and just report the
> files created.

Then summarize for the user:
- The directory tree you created under `gitops/sre-tools/<tool>/`.
- Any secrets they still need to fill in and encrypt.
- The next commands to deploy: `cd gitops/sre-tools/<tool> && ./deploy.sh`.
- Remind them to commit the new directory and open a PR themselves (this skill does not
  commit or push).

---

## Key rules

- **Always use the AskUserQuestion tool** for questions — never ask in plain text.
- **Read the existing tools first** and match their exact conventions; do not invent structure.
- **`deploy.sh` must** start with `#!/usr/bin/env bash` + `set -euo pipefail`, define a
  `NAMESPACE` variable, and reference `"$NAMESPACE"` in the helm command.
- **Never write plaintext secrets** into the repo. Generate encrypted `secrets-encrypted.yaml`
  only via `helm secrets encrypt`, run by the user.
- **Scaffold only** — do not commit, push, open a PR, or run helm against the cluster unless
  the user explicitly asks.
- **Company-wide exposure = Tailscale** (`https://<hostname>.baiji-wall.ts.net`);
  internal routing = Traefik IngressRoute; otherwise ClusterIP + port-forward.
