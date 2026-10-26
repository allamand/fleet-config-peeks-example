# fleet-config-peeks-example

Example **fleet repo** for a **non-workshop, direct** deployment of
[`aws-samples/appmod-blueprints`](https://github.com/aws-samples/appmod-blueprints)
with the `kind-kro-ack` cluster provider. You point your install at this repo via
`fleetRepo.url` in `config.local.yaml`, and ArgoCD on the self-managed hub reconciles
the spoke clusters declared here.

This repo ships **two spokes pre-committed** — `dev` and `prod` — under
`gitops/fleet/spoke-values/tenants/default/kro-clusters/`. That is the **zero-PAT path**
(Case A below): because the spoke declarations are already in the repo, ArgoCD only has
to **read** them, which it does over an anonymous clone of a public repo. **No write
credential / Personal Access Token is required** for this path.

> This is an **example**. Every value is generic: no AWS account IDs, no tokens, no
> internal hostnames, no real domains, no secret values. Placeholders are written as
> `<org>`, `<account-id>`, `<secret-name>`.

---

## Repository layout

```
gitops/
  fleet/
    hub/
      values.yaml                 # hub (control-plane) self-registration
    members/                      # fleet membership -> ArgoCD cluster secrets
      dev/values.yaml             #   spoke "dev"  (mode: externalSecret)
      prod/values.yaml            #   spoke "prod" (mode: externalSecret)
    spoke-values/
      tenants/
        default/                  # tenant (neutral default, never "workshop")
          kro-clusters/
            clusters/
              dev.json            # PRE-COMMITTED trigger  {"clusterName":"dev"}
              prod.json           # PRE-COMMITTED trigger  {"clusterName":"prod"}
            values.yaml           # per-spoke specs (region, cidr, k8sVersion, ...)
configs/
  multi-acct/
    values.yaml                   # ACK CARM: spoke namespace -> account (placeholder)
overlays/
  clusters/
    dev/platform-manifests/values.yaml   # per-cluster overlay (huggingfaceModels: [])
    prod/platform-manifests/values.yaml
```

### How ArgoCD reads this

The hub's `clusters-kro` ApplicationSet
(`gitops/bootstrap/clusters-kro.yaml` in appmod-blueprints) globs

```
fleet/spoke-values/tenants/*/kro-clusters/clusters/*.json
```

— one trigger JSON **per cluster** — and for each one reads the matching per-spoke
spec from `tenants/<tenant>/kro-clusters/values.yaml`. The `fleet-secrets` ApplicationSet
globs `fleet/members/*/values.yaml` to register each spoke with ArgoCD. The JSON is
intentionally minimal (`{"clusterName":"<name>"}`); the real spec lives in `values.yaml`.

**`accountId` is deliberately absent** from every spec here: the chart defaults it to
`global.accountId`, which the hub propagates from its own cluster-secret (your real
account) at reconcile time — so no account number ever has to live in a public repo.

---

## Case A — pre-filled public repo (zero-PAT, recommended default)

**When:** you want the quickstart with the fewest moving parts, and you are happy for the
spoke declarations to be public.

**How it works:** `fleetRepo.url` points at **this public repo** (your fork or this one).
The spokes are already committed in `clusters/*.json` + `kro-clusters/values.yaml`. ArgoCD
clones the public repo **read-only** (anonymous — no credential), sees the declarations,
and reconciles `dev` and `prod`. **No PAT, no Secrets Manager entry, nothing to authorize.**

```yaml
# config.local.yaml (extract)
clusterProvider: kind-kro-ack
fleetRepo:
  url: https://github.com/<org>/fleet-config-peeks-example.git
  revision: main
  basepath: ""
```

**To add or change a spoke**, commit directly to the repo:

1. add `gitops/fleet/spoke-values/tenants/default/kro-clusters/clusters/<name>.json`
   containing `{"clusterName": "<name>"}`;
2. add a `<name>:` block under `clusters:` in
   `gitops/fleet/spoke-values/tenants/default/kro-clusters/values.yaml`
   (a disjoint `cidr.vpcCidr`, `region`, `k8sVersion`, ...);
3. add `gitops/fleet/members/<name>/values.yaml` (copy `dev/`);
4. `git commit && git push` to your copy of this repo.

ArgoCD picks the change up on its next sync. Editing the public repo yourself is the whole
workflow — there is no installer push, so **still no PAT**.

---

## Case B — generative (`spokes:apply`, **PAT required**)

**When:** you would rather **declare** spokes in `config.local.yaml` and let the installer
**materialise and push** the `clusters/*.json` for you, instead of hand-editing the repo.

**How it works:** you declare a `spokes:` map in `config.local.yaml`; at install time the
`spokes:apply` task clones the fleet repo, writes the trigger JSON + values entry + member
file for each declared spoke, commits, and **pushes**. Because it **writes** to the repo, it
needs a **write credential** — a Git Personal Access Token (PAT).

```yaml
# config.local.yaml (extract)
clusterProvider: kind-kro-ack
fleetRepo:
  url: https://github.com/<org>/<your-fleet-repo>.git   # a repo YOU can push to
  revision: main
  basepath: ""
spokes:
  dev:
    region: us-west-2
    vpcCidr: 10.1.0.0/16
    tenant: default
    kubernetesVersion: "1.31"
    autoMode: true
  prod:
    region: us-west-2
    vpcCidr: 10.2.0.0/16
    tenant: default
    kubernetesVersion: "1.31"
    autoMode: true
```

> **Schema vs fleet-repo field names.** The `spokes:` map above is the **stage-1 config**
> shape (`vpcCidr`, `kubernetesVersion`, `autoMode`). When `spokes:apply` writes into the
> fleet repo it emits the **chart** shape used by `kro-clusters/values.yaml` in this repo:
> `vpcCidr` -> `cidr.vpcCidr` (+ the subnet split), `kubernetesVersion` -> `k8sVersion`,
> and `autoMode` maps onto the chart's `enableGeneralPurposeNodePool` knob (Auto Mode
> itself is a hub/RGD default). The pre-committed files in this repo are already in that
> fleet-repo shape.

### The PAT

- **Scope — minimal:**
  - public fleet repo → `public_repo`
  - private fleet repo → `repo`
- **Where it lives:** in **AWS Secrets Manager**, as a dedicated secret. Reference the
  secret by **name/ARN** in your config (a **placeholder** — `<secret-name>` — never the
  token value). **Never** paste a PAT into config, into this repo, or into chat.
- **Who reads it, and when:**
  - `spokes:apply` (stage 2) reads it **at push time** — clone → write JSONs → commit → push;
  - **ArgoCD** reads it **continuously** to reconcile (only needed if the repo is private;
    a public repo is read anonymously).
- **The config field that references it:** the Secrets Manager secret name/ARN goes in the
  config, not the token. The secret holds the git credential (token); the install task and
  ArgoCD resolve it at runtime.

---

## Case C — private fleet repo

**When:** your fleet repo must not be public at all.

**How it works:** `fleetRepo.url` is a **private** repo. A credential is now needed for
**both** directions — ArgoCD to **read/clone** and reconcile, and (if you use Case B)
`spokes:apply` to **push**. Two credential sources, pick one:

- **Reuse the ArgoCD-registered repo credential** `argocd/fleet-config-repo` — the platform
  is consumer-agnostic: it adopts the repo credential **you** wired, it does not mint an
  identity. ArgoCD already holds enough to read/reconcile the fleet repo; the same secret can
  serve `spokes:apply`'s clone/push.
- **Or a dedicated AWS Secrets Manager secret** `<secret-name>` (same mechanics as Case B) —
  referenced by name/ARN in config, read by `spokes:apply` at push and by ArgoCD continuously.

```yaml
# config.local.yaml (extract)
fleetRepo:
  url: https://github.com/<org>/<private-fleet-repo>.git
  revision: main
  basepath: ""
# plus the credential reference (ArgoCD argocd/fleet-config-repo, or a dedicated
# Secrets Manager secret named <secret-name>) — never the token value itself.
```

---

## Honest note on what "public" removes

Making the fleet repo public removes the credential for the **read / clone** path — ArgoCD
reconciling, the hub-only bootstrap, and reading the example overlays — **and** for the
**pre-filled Case A** path (no push at all). It does **not** remove the credential for a
**generative push** of spokes: anything that **writes** `clusters/<name>.json` back to the
repo (Case B `spokes:apply`, or a private repo in Case C) still needs a write credential.

In short: **public = no creds to read** (hub + pre-committed spokes); **a PAT is still
needed to push** generated spokes.
