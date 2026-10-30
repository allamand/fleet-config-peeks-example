# fleet-config-peeks-example

Example **fleet repo** for a **non-workshop, direct** deployment of
[`aws-samples/appmod-blueprints`](https://github.com/aws-samples/appmod-blueprints)
with the `kind-kro-ack` cluster provider. You point your install at this repo via
`fleetRepo.url` in `config.local.yaml`, and ArgoCD on the self-managed hub reconciles
the spoke clusters declared here.

This repo is shared by **two hubs** (`resource_prefix` `peeks-a` and `peeks-b`) and
ships **one spoke per hub pre-committed** — `peeks-a-dev` and `peeks-b-dev` — under
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
      peeks-a/peeks-a-dev/values.yaml   # spoke "peeks-a-dev" of hub peeks-a (mode: externalSecret)
      peeks-b/peeks-b-dev/values.yaml   # spoke "peeks-b-dev" of hub peeks-b (mode: externalSecret)
    spoke-values/
      tenants/
        default/                  # tenant (neutral default, never "workshop")
          kro-clusters/
            clusters/
              peeks-a/peeks-a-dev.json   # PRE-COMMITTED trigger {"clusterName":"peeks-a-dev"}
              peeks-b/peeks-b-dev.json   # PRE-COMMITTED trigger {"clusterName":"peeks-b-dev"}
            values.yaml           # per-spoke specs (region, cidr, k8sVersion, ...)
  overlays/
    environments/
      control-plane/enabled-addons.yaml  # CORE addon set of the hubs
      dev/enabled-addons.yaml            # CORE addon set of every spoke labelled environment: dev
      prod/enabled-addons.yaml           # CORE addon set of every spoke labelled environment: prod
    tenants/
      default/enabled-addons.yaml        # CORE tenant-wide addons of every spoke labelled tenant: default (cert-manager)
configs/
  multi-acct/
    values.yaml                   # ACK CARM for the hub only (spokes: see kro-clusters)
overlays/
  environments/
    control-plane/enabled-addons.yaml            # AGENT / OAP layer of the hubs
  clusters/
    peeks-a-hub/ingress-class-alb/values.yaml    # hub ALB source-IP allowlist
```

### How ArgoCD reads this

The hub's `clusters-kro` ApplicationSet
(`gitops/bootstrap/clusters-kro.yaml` in appmod-blueprints) globs one trigger JSON
**per cluster**, then reads the matching per-spoke spec from
`tenants/<tenant>/kro-clusters/values.yaml`. The `fleet-secrets` ApplicationSet globs the
member files to register each spoke with ArgoCD. The JSON is intentionally minimal
(`{"clusterName":"<name>"}`); the real spec lives in `values.yaml`.

Several hubs share this repo on one branch. Each hub's cluster-secret carries
`fleet_scope_by_prefix: "true"`, which inserts its `resource_prefix` into both globs:

```
fleet/spoke-values/tenants/*/kro-clusters/clusters/<resource_prefix>/*.json
fleet/members/<resource_prefix>/*/values.yaml
```

so hub `peeks-a` only sees `peeks-a-dev` and hub `peeks-b` only sees `peeks-b-dev`.
Without that annotation (single-hub default) the globs are `clusters/*.json` and
`members/*/values.yaml`, and the files sit directly under `clusters/` and `members/`.

Per-cluster overlays (`overlays/clusters/<cluster>/<addon>/values.yaml`) are read only for
clusters a hub has registered, so `peeks-a-hub/*` only affects hub `peeks-a`.

**`accountId` is deliberately absent** from every spec here: the chart defaults it to
`global.accountId`, which the hub propagates from its own cluster-secret (your real
account) at reconcile time — so no account number ever has to live in a public repo.

---

## Case A — pre-filled public repo (zero-PAT, recommended default)

**When:** you want the quickstart with the fewest moving parts, and you are happy for the
spoke declarations to be public.

**How it works:** `fleetRepo.url` points at **this public repo** (your fork or this one).
The spokes are already committed in `clusters/<resource_prefix>/*.json` +
`kro-clusters/values.yaml`. ArgoCD clones the public repo **read-only** (anonymous — no
credential), sees the declarations, and each hub reconciles its own spoke (`peeks-a-dev`,
`peeks-b-dev`). **No PAT, no Secrets Manager entry, nothing to authorize.**

```yaml
# config.local.yaml (extract)
clusterProvider: kind-kro-ack
fleetRepo:
  url: https://github.com/<org>/fleet-config-peeks-example.git
  revision: main
  basepath: ""
```

**To add or change a spoke**, commit directly to the repo:

1. add `gitops/fleet/spoke-values/tenants/default/kro-clusters/clusters/<resource_prefix>/<name>.json`
   containing `{"clusterName": "<name>"}` (single hub without `fleet_scope_by_prefix`:
   directly under `clusters/`);
2. add a `<name>:` block under `clusters:` in
   `gitops/fleet/spoke-values/tenants/default/kro-clusters/values.yaml`
   (a disjoint `cidr.vpcCidr`, `region`, `k8sVersion`, ...);
3. add `gitops/fleet/members/<resource_prefix>/<name>/values.yaml` (copy
   `peeks-a/peeks-a-dev/`; single hub: `members/<name>/values.yaml`);
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
    kubernetesVersion: "1.37"
    autoMode: true
  prod:
    region: us-west-2
    vpcCidr: 10.2.0.0/16
    tenant: default
    kubernetesVersion: "1.37"
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
  - public fleet repo → classic `public_repo` (fine-grained: Contents Read-only)
  - private fleet repo → classic `repo` (fine-grained: Contents Read and write)
- **Where it lives:** in **AWS Secrets Manager**, as the property `git_token` of the
  per-hub secret `<hub-cluster-name>/secrets` (the exact `remoteRef.property` the platform
  resolves), or a dedicated secret `<secret-name>` you point an ExternalSecret at. Store
  only the **token** — a **placeholder** (`<PAT>`) here, **never** the real value in
  config, in this repo, or in chat.
- **Who reads it, and when:**
  - the generative spoke push (stage 2) reads it **at push time** — clone → write JSONs → commit → push;
  - **ArgoCD** reads it **continuously** to reconcile (only needed if the repo is private;
    a public repo is read anonymously).
- **How it reaches the tooling — not through config.** There is **no credential field in
  `config.local.yaml`** (`config.schema.json` exposes only `fleetRepo.{url,revision,basepath}`).
  The token is surfaced as the ArgoCD `repository` Secret `argocd/fleet-config-repo`
  (created directly or projected from Secrets Manager by ESO — see Case C), which both
  ArgoCD and the provider's push read at runtime. The config carries only the repo URL.

---

## Case C — private fleet repo

**When:** your fleet repo must not be public at all.

**How it works:** `fleetRepo.url` is a **private** repo, so a credential is now needed
for **both** directions — ArgoCD to **read/clone** and reconcile, and the provider's
own GitOps push (hub self-adoption / any generative spoke push) to **write**. There is
**no credential field in the config** — the config carries only `fleetRepo.url`,
`revision`, `basepath` (`config.schema.json`, and the provider reads exactly
`yq '.fleetRepo.url // .repo.url'` — nothing resolves a token from config). The
credential is a **Kubernetes Secret ArgoCD recognises**, named **`fleet-config-repo` in
the `argocd` namespace**, with the ArgoCD `repository` label. The provider reads that
same secret back for its own clone/push, and **refuses** the pivot if it is absent.

Below are the three concrete things you must do. Use placeholders only — **never put a
real token in config, in this repo, or in chat.**

### (a) Store the PAT — minimal scope, where it lives

Mint a Git Personal Access Token that can **read the private repo** (and push, if you
let the installer generate spokes):

- **GitHub classic PAT:** scope `repo` (a classic PAT has no read-only tier for a
  private repo). For a **public** fleet repo only `public_repo` is needed — but a public
  repo is Case A/B, not Case C.
- **GitHub fine-grained PAT:** repository access limited to the one fleet repo, permission
  **Contents: Read and write** (Read-only if ArgoCD only ever reads and you never push
  generated spokes).

Keep the token in **AWS Secrets Manager**, not on disk. The platform's own ESO wiring
reads the token from the property **`git_token`** of the per-hub secret
**`<hub-cluster-name>/secrets`** (that is the exact `remoteRef.property` the chart uses).
So either reuse that key, or create a dedicated secret and point your own ExternalSecret at it:

```bash
# Option 1 — add git_token to the hub's existing <hub>/secrets (merge, don't clobber):
aws secretsmanager get-secret-value --secret-id "<hub-cluster-name>/secrets" \
  --query SecretString --output text \
  | jq --arg t "<PAT>" '. + {git_token: $t}' \
  | aws secretsmanager put-secret-value --secret-id "<hub-cluster-name>/secrets" \
      --secret-string file:///dev/stdin

# Option 2 — a dedicated secret you reference from your own ExternalSecret (see (b)):
aws secretsmanager create-secret --name "<secret-name>" \
  --secret-string '{"git_token":"<PAT>"}'
```

> The platform never stores the repo **URL or username** in Secrets Manager — only the
> token (`git_token`). The URL/username are supplied when the ArgoCD secret is rendered
> (see (b)). `<PAT>` and `<secret-name>` are placeholders.

### (b) Create the ArgoCD `repository` secret

ArgoCD authenticates a private repo clone from an Opaque Secret in the `argocd`
namespace carrying the label `argocd.argoproj.io/secret-type: repository` and
`url`/`username`/`password` (`password` = the PAT). This is exactly the shape the
platform itself seeds. Two ways:

**Direct (`kubectl apply`)** — simplest, token is set once at apply time:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: fleet-config-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/<org>/<private-fleet-repo>.git
  username: <org>          # any non-empty value; GitHub ignores it when a PAT is the password
  password: <PAT>          # the Personal Access Token — placeholder, never the real value
```

**Or via ESO from Secrets Manager** — the token self-heals on rotation (this mirrors the
platform's own `argocd-gitlab-repo-creds` chart, which projects `git_token` → the ArgoCD
`repository` secret):

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: fleet-config-repo
  namespace: argocd
spec:
  secretStoreRef:
    name: aws-secrets-manager      # your ClusterSecretStore
    kind: ClusterSecretStore
  refreshInterval: "5m"
  target:
    name: fleet-config-repo
    creationPolicy: Owner
    template:
      engineVersion: v2
      metadata:
        labels:
          argocd.argoproj.io/secret-type: repository
      data:
        type: git
        url: "https://github.com/<org>/<private-fleet-repo>.git"
        username: "<org>"
        password: "{{ .GIT_PASSWORD }}"
  data:
    - secretKey: GIT_PASSWORD
      remoteRef:
        key: <secret-name>          # or <hub-cluster-name>/secrets
        property: git_token
```

### (c) Reference it from the config — you don't

**There is nothing to add to `config.local.yaml` for the credential.** The config block
stays exactly as in Case A/B — only the URL changes to the private repo:

```yaml
# config.local.yaml (extract)
fleetRepo:
  url: https://github.com/<org>/<private-fleet-repo>.git
  revision: main
  basepath: ""
# No credential key here. The private fleet repo is wired entirely by the ArgoCD
# Secret argocd/fleet-config-repo (created in (b)) — directly, or via the ESO that
# projects it from Secrets Manager. config.schema.json exposes only
# fleetRepo.{url,revision,basepath}; it has no token/secret field.
```

**Who reads what, and when:**

- **ArgoCD** reads `argocd/fleet-config-repo` **continuously** to clone and reconcile the
  private fleet repo (this is the only credential a private repo needs just to run).
- The **kind-kro-ack provider** reads that *same* `argocd/fleet-config-repo` secret
  (`username`/`password`) **at pivot/push time** for its own GitOps writes, and **refuses
  with an explicit error if the secret is absent** — it never mints an identity or builds
  a URL of its own; it reuses the credential **you** registered.

---

## Honest note on what "public" removes

Making the fleet repo public removes the credential for the **read / clone** path — ArgoCD
reconciling, the hub-only bootstrap, and reading the example overlays — **and** for the
**pre-filled Case A** path (no push at all). It does **not** remove the credential for a
**generative push** of spokes: anything that **writes** `clusters/<name>.json` back to the
repo (Case B `spokes:apply`, or a private repo in Case C) still needs a write credential.

In short: **public = no creds to read** (hub + pre-committed spokes); **a PAT is still
needed to push** generated spokes.

---

## Addon enablement (`enabled-addons` overlays)

Everything above is about **which spokes** exist and **how the repo is read**. This
section is about **which addons run on the hub** (control-plane) itself. That set is
driven entirely by **two `enabled-addons.yaml` files in this repo** — there is **no RGD
and no annotation** involved in choosing them.

### How an addon gets turned on

Both files are read by the hub's **`hub-fleet-secret` ApplicationSet**, which merges them
into **`.Values.enabledAddons`** and stamps, for every key set to `true`, an
**`enable_<addon>` label on the hub's cluster-secret**. The platform's ApplicationSets
select on those labels, so the label is the switch: `foo: true` → `enable_foo` label
present → the `foo` addon's Application syncs. Flip it to `false` → the label is gone →
the addon is not deployed. The value flows strictly from these repo files; nothing is
decided by a ResourceGraphDefinition or a live `kubectl annotate` (an annotation would be
reverted on the next reconcile — the repo file is the source of truth).

### The two slots — exact paths and why there are two

The `hub-fleet-secret` appset reads **two** overlay slots and merges them, so the two
files live at **different paths** and are **not interchangeable**:

| Layer | Slot | basepath | Path in this repo |
|-------|------|----------|-------------------|
| **CORE** (agent-free) | `$values` | `gitops/` | `gitops/overlays/environments/control-plane/enabled-addons.yaml` |
| **AGENT / OAP** | `$overlay` | `""` (repo root) | `overlays/environments/control-plane/enabled-addons.yaml` |

- **CORE** (`$values`, basepath `gitops/`) is **always read**. It carries the
  **agent-free** platform: crossplane, backstage, keycloak, kargo, the `argo_*` stack,
  external-secrets / external-dns, observability, ACK/ML knobs, etc. This is the layer
  that **unblocks the spokes and the platform addons**. The base solution is kept
  agent-free here on purpose.
- **AGENT / OAP** (`$overlay`, basepath `""`, i.e. the **repo root**) is merged **last**,
  **on top** of CORE. It carries the Open Agentic Platform layer:
  `agent_platform`, `bifrost`, `agent_gateway`, `oam_components`, `otel_collector`,
  `langfuse`, `agent_core`, `amp_incident`, `aiops_agent`. It is a **separate slot** so
  the generic base stays agent-free and the agent layer can be wired in (or left out)
  independently.

The two slots are distinct precisely so you can run the full platform **without** the
agent layer by simply not wiring the overlay slot — the CORE file stands alone.

### Enabling / disabling an addon

1. Open the `enabled-addons.yaml` for the **right slot** (CORE for platform addons, AGENT
   for the OAP keys above).
2. Set the addon's key to `true` (enable) or `false` (disable).
3. `git commit && git push` to your copy of this repo.
4. ArgoCD reconciles: the `enable_<addon>` label appears/disappears on the hub
   cluster-secret and the addon's Application is created/removed.

### Important: the AGENT layer only sets labels — OAP is **double-gated**

Enabling the agent keys in the AGENT overlay **only stamps the `enable_*` labels**. It
does **not**, by itself, deploy the Open Agentic Platform. OAP is actually applied **only
when the `agenticRepo` block is present in `config.local.yaml`** — that block is the gate
for the `agentic:bootstrap-direct` task, which is what `kubectl apply`s the OAP
`agent-platform-app`.

So there are **two independent conditions**, and you need **both** for OAP to run:

- **AGENT overlay keys = `true`** → the `enable_*` labels are present (this file), **and**
- **`agenticRepo` set in `config.local.yaml`** → `agentic:bootstrap-direct` applies the
  OAP `agent-platform-app`.

If the AGENT labels are on but **`agenticRepo` is absent from the config**, the labels are
stamped but **OAP is never applied** — this is **harmless** (the selectors find no OAP
Applications to sync; nothing breaks). Conversely, `agenticRepo` without the AGENT labels
leaves the OAP ComponentDefinitions/selectors unlabelled. Set **both** to actually get the
agent platform.

### The chain, end to end

```
enabled-addons.yaml (this repo, 2 slots)
   ├─ CORE  : gitops/overlays/environments/control-plane/enabled-addons.yaml   ($values, basepath gitops/)
   └─ AGENT : overlays/environments/control-plane/enabled-addons.yaml          ($overlay, basepath "")
        │
        ▼
   hub-fleet-secret ApplicationSet   (merges both into .Values.enabledAddons)
        │
        ▼
   enable_<addon> labels on the hub cluster-secret
        │
        ├─ platform/agent Applications select on the labels and sync
        │
        └─ (OAP only) agent-platform-app is applied by agentic:bootstrap-direct
                         ─ IF AND ONLY IF agenticRepo is set in config.local.yaml
```

### Spoke addons: one file per environment

Spokes follow the same rule as the hub, through the `fleet-secrets` ApplicationSet
instead of `hub-fleet-secret`. A spoke's addon set is chosen by its **environment**, not by
its name:

```
fleet/members/<prefix>/<cluster>/values.yaml   labels.environment: dev
        │
        ▼
gitops/overlays/environments/dev/enabled-addons.yaml   ($values, basepath gitops/)
        │
        ▼
enable_<addon> labels on the spoke cluster-secret (fleet-secret chart, ESO)
```

- `gitops/overlays/environments/dev/enabled-addons.yaml` and `.../prod/enabled-addons.yaml`
  are verbatim copies of the appmod-blueprints files of the same path. A separate fleet repo
  does **not** inherit them from appmod: if the file for an environment is missing here,
  every spoke of that environment registers with **zero** addons, silently.
- Every hub that uses this repo reads the same files, so `peeks-a-dev` and `peeks-b-dev` get
  the same set. To change one spoke only, add
  `gitops/overlays/clusters/<cluster>/addon-overrides.yaml` with an `enabledAddons:` map
  (`false` turns a key off).
- Keep `environment` identical in the member file (`labels.environment`, which picks the
  `enabled-addons.yaml`) and in the spoke's entry in
  `fleet/spoke-values/tenants/<tenant>/kro-clusters/values.yaml` (which becomes the
  `environment` label of the cluster-secret and picks the per-addon values under
  `overlays/environments/<env>/`). The kro-clusters chart defaults a missing `environment`
  to `staging`.
- A spoke also reads `gitops/overlays/tenants/<tenant>/enabled-addons.yaml`, keyed by
  `labels.tenant` in its member file, **before** the environment file (environment and
  per-cluster files can still override it). `tenants/default/` enables `cert_manager`,
  which the AWS Load Balancer Controller and KubeVela webhooks require; it mirrors
  appmod-blueprints `tenants/workshop/`. Without it, `aws-load-balancer-controller`,
  `kubevela` and `ingress-class-alb` stay OutOfSync on the spokes.
- Hubs are unaffected: they read only `control-plane`.
