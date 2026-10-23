# Runners: Where and How Lifecycle Scripts Execute

Applies to every component with `runnerImage` + `deploy`/`start`/`stop`/`destroy`
scripts: `GenericComponent`, `Terraform`, `Helm`, `KubernetesManifest`.

## TL;DR

- Scripts do **not** run inside the environment. Each script run is a one-off Pod in its own
  throwaway namespace, on the **runner cluster**: the Bunnyshell-managed jobs cluster by default,
  or a self-hosted cluster if the project/environment is configured with one.
- With the **managed runner** (the default), the runner **cannot reach the environment's pods or
  services** by Kubernetes DNS (`my-service`, `my-service.env-abc123.svc.cluster.local`): it is a
  different cluster, and private-IP egress is blocked. Even on a self-hosted runner, short names
  like `my-service` never resolve (different namespace).
- To work with a private in-environment service, run the work **inside the environment's
  namespace** (see [Networking](#networking)) instead of exposing the service publicly.
- `Helm` and `KubernetesManifest` runners get a **kubeconfig for the environment's cluster**
  (`KUBECONFIG` is set), so `kubectl`/`helm` work against it. `GenericComponent` and
  `Terraform` runners do **not**.
- `deploy` re-runs on **every** environment deploy. On environment **start/stop** only the
  `start`/`stop` scripts run, and only if defined. Anything `deploy` set up inside a workload
  that loses its state on restart (e.g. an in-memory emulator) is **not** re-applied unless
  `start` does it too.

## Where the runner runs

| Setting (Project or Environment → deployment runner) | Runner cluster |
|---|---|
| Default / "managed runner" | Bunnyshell-managed shared jobs cluster |
| A connected Kubernetes integration (self-hosted runner) | That cluster |

Environment-level setting wins; if unset it inherits the Project's. The environment's own
cluster is only the **deploy target**, never where the script runs, unless a self-hosted
runner is explicitly set to that same cluster.

Per run, Bunnyshell creates:

- Namespace `neo-runner-job-<orgUnique>-<jobId>`, created fresh for the run and garbage-collected
  a few hours later.
- A bare Pod `deployment-runner` (`restartPolicy: Never`, no service-account token):
  - init container `context-builder`: clones the git repo and stages scripts + helpers
  - container `runner`: **your `runnerImage`** (`imagePullPolicy: Always`) running the steps
  - sidecar `monitor`: streams logs, enforces the timeout, reports results/exported variables

## Networking

**Managed runner (default):** network policies allow egress **only to public IPs**
(RFC1918 ranges 10/8, 172.16/12, 192.168/16 and the cloud metadata IP are blocked).

| Managed runner wants to reach | Works? | How |
|---|---|---|
| Public internet / cloud APIs (AWS, GCP, Azure, registries) | Yes | directly |
| An env component via its public ingress host | Yes | `https://{{ components.<name>.ingress.hosts[0].hostname }}` (component must declare `hosts:`) |
| An env component by service DNS / cluster IP | **No** | different cluster + private egress blocked |
| Env cluster's Kubernetes API (Helm / KubernetesManifest) | Yes | injected `KUBECONFIG` (`kubectl apply`/`exec`/`wait`) |
| Env cluster's Kubernetes API (GenericComponent / Terraform) | No kubeconfig provided | prefer a KubernetesManifest/Helm component instead (see below) |

**Self-hosted runner:** Bunnyshell applies no egress policies; what is reachable depends on the
customer's own network and policies. If it is the environment's own cluster, in-cluster DNS can
work but must be namespace-qualified (`my-service.{{ env.k8s.namespace }}.svc.cluster.local`),
because the runner is in its own namespace.

If a step must talk to a **private** in-environment service (seeding a DB, configuring an
emulator such as Azurite/LocalStack/MinIO), prefer, in this order:

1. **Run the work inside the environment's namespace** from a `KubernetesManifest` component:
   it gets the env cluster's kubeconfig, so it can `kubectl apply` a one-off `Job` (or
   `kubectl exec` into the target pod) where `my-service:port` resolves. See the example below.
2. Do it from the workload itself: an `InitContainer`/`SidecarContainer` or the container's own
   entrypoint/command.
3. Only if the service is meant to be public anyway: call its ingress host from the runner.
   Do **not** expose a private data service (databases, emulators with default/well-known
   keys) just so a runner can reach it; if unavoidable, restrict the host with `allowedIps`
   and use non-default credentials.

Avoid `kubectl port-forward` in steps: it blocks until killed, so a bare one hangs the step
until the runner timeout. If you must, background it, wait for readiness, and `kill` it.

## Inside the runner container

| Path / variable | What |
|---|---|
| `/bns/context` | Working directory; the `gitRepo` checkout at the deployed commit (**whole repo**; `.git` is removed, so `git` commands do not work) |
| `gitApplicationPath` | Not used by the runner (it only scopes change detection for auto-update). `cd` into subfolders yourself |
| `/bns/helpers/helm/bns_post_renderer` | Helm post-renderer (adds Bunnyshell labels + image pull secrets; needs `kubectl` in the image) |
| `/bns/helpers/kubernetes/bns_kustomize_init` | Kustomize label init (needs `kustomize`) |
| `/bns/helpers/terraform/get_managed_backend` | Prints a managed S3 backend block |
| `KUBECONFIG=/bns/helpers/kubeconfig` | Always exported; the file exists only for Helm/KubernetesManifest. Its default namespace is the env namespace |
| Env vars | Project vars → environment vars → `environmentVariablesGroups` → component `environment` (later wins), secrets decrypted, interpolated |
| `BNS_TF_STATE_*` | Terraform only: managed state backend credentials/bucket |
| `BNS_IMAGE_PULL_SECRETS` | Helm/KubernetesManifest only: comma-separated pull secret names |

`{{ ... }}` interpolation (e.g. `{{ env.k8s.namespace }}`, `{{ env.unique }}`) is resolved
**before** the scripts are written, so it is plain text by the time the shell sees it.

Image requirements: `/bin/sh` and basic coreutils (`sed`, `cut`, `tail`, `base64`, `tr`, `cat`,
`cp`, `chmod`, `sleep`). Busybox/Alpine is fine; bash is not required (and not used). The image's
default user is used. `runnerImage: '@self'` (the GenericComponent default) uses the component's
own built image.

Resources: default requests 250m CPU / 300Mi, limits 1 CPU / 2Gi. Default timeout 30 minutes.
CPU/memory/timeout overrides apply to self-hosted runners only.

## How steps are executed

Each list item under `deploy`/`start`/`stop`/`destroy` becomes its own script file. A wrapper,
run with **`/bin/sh` and `set -u`** (no `set -e`), **sources** them one after another in the
same shell, from `/bns/context`, with `exit` aliased to `return`.

Consequences:

- Variables, `export`s and `cd` **carry over** between list items.
- A step fails only if its **last command** exits non-zero; earlier failures in a multi-line
  step are ignored unless you add `set -e` (which then also applies to later steps, since the
  shell is shared) or `&&`/`|| exit 1`.
- The first failing step stops the run; later steps do not execute.
- `set -u` is active: referencing an **unset** variable fails the step. Use `${VAR:-}` for
  optional ones.
- Shebangs are ignored; bash-only syntax (`[[ ]]`, arrays, `source`, `pipefail`) breaks on
  dash/busybox `sh`. Use POSIX sh, or call `bash -c '...'` / a `.sh` file with `bash` explicitly.
- Do not hide errors with `2>/dev/null` while debugging; step output is what appears in the
  pipeline logs.

## exportVariables

Only captured from **`deploy`**. After the last step, each listed name is read as a **shell
variable** from the shared shell, so a plain assignment is enough:

```yaml
deploy:
  - 'SERVICE_URL=$(gcloud run services describe svc --format "value(status.url)")'
exportVariables:
  - SERVICE_URL
```

Consumers use `{{ components.<name>.exported.SERVICE_URL }}` (value from the latest successful
deploy). Max ~4 MB per value.

## When each script runs

| Script | Runs on | Notes |
|---|---|---|
| `deploy` | Every environment deploy that includes the component (full deploy, or partial deploy selecting it/its dependents) | No "skip if unchanged". Must be **idempotent** |
| `start` | Environment start (incl. after auto-stop / scheduled stop) | Skipped if empty. `deploy` does **not** run on start |
| `stop` | Environment stop | Skipped if empty |
| `destroy` | Environment delete / component removal | Bunnyshell does not track resources a script created. Clean them up here |

`dependsOn` orders runner jobs after other components' jobs. It does **not** put the runner
on the environment's network.

## Logs

Step output streams live into the pipeline job logs (`bns pipeline logs --id <PIPELINE_ID>`,
or the job in the UI), and the full log is stored after the run ("runner job results").
The runner namespace is short-lived, so `kubectl` on the environment's cluster will **not** find
the runner pod.

## Example: configure a private emulator (Azurite CORS) from inside the environment

Wrong: a `GenericComponent` using `http://azurite:10000` (managed runner: never resolves), or
exposing Azurite publicly with its well-known dev key just so the runner can reach it.

Right: a repo-less `KubernetesManifest` component launches a one-off `Job` in the environment
namespace, where `azurite:10000` resolves, waits for it, and does the same on `start` because
`deploy` does not re-run then. The Job's own retries (`backoffLimit`) cover Azurite still
starting up.

```yaml
- kind: KubernetesManifest
  name: azurite-cors-init
  runnerImage: 'alpine/k8s:1.31.0'
  dependsOn: [azurite]
  deploy: &cors_steps
    - |
      cat <<'EOF' > job.yml
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: azurite-cors-init
      spec:
        backoffLimit: 10
        ttlSecondsAfterFinished: 600
        template:
          spec:
            restartPolicy: OnFailure
            containers:
              - name: az
                image: mcr.microsoft.com/azure-cli:2.67.0
                env:
                  - name: CS
                    # Azurite's documented local-dev account; only reachable inside the namespace
                    value: 'DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://azurite:10000/devstoreaccount1;'
                command: ['sh', '-ec']
                args:
                  - |
                    az storage cors clear --services b --connection-string "$CS" --only-show-errors
                    az storage cors add --services b --origins 'https://app-{{ env.unique }}.example.com' \
                      --methods GET HEAD POST PUT DELETE OPTIONS --allowed-headers '*' --exposed-headers '*' \
                      --max-age 3600 --connection-string "$CS" --only-show-errors
                    az storage cors list --services b --connection-string "$CS" -o table
      EOF
    - 'kubectl delete job azurite-cors-init --ignore-not-found'
    - 'kubectl apply -f job.yml'
    - |
      kubectl wait --for=condition=complete --timeout=600s job/azurite-cors-init
      rc=$?
      kubectl logs job/azurite-cors-init --tail=50
      [ "$rc" -eq 0 ]
  start: *cors_steps   # Azurite without a volume loses CORS while stopped; deploy doesn't re-run on start
  destroy:
    - 'kubectl delete job azurite-cors-init --ignore-not-found'
```

Notes: YAML anchors/aliases (`&cors_steps` / `*cors_steps`) are supported; Bunnyshell expands
them when parsing, so an exported definition shows the steps repeated. The kubeconfig's default namespace is the environment's, so no `-n` is needed. The last
step ends with `[ "$rc" -eq 0 ]` so the step fails when the Job does, while still printing its logs.
Pin image versions you have tested.
