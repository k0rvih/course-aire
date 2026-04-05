# course-aire

Materials for **AI Reliability Engineering**: a local Kubernetes lab on Oracle VirtualBox (Vagrant), with **Argo CD** GitOps for agents, gateways, and supporting workloads.

> **Warning — insecure lab defaults**
> This setup trades security for convenience. **Do not expose these VMs or UIs to the internet or untrusted networks**, and do not reuse these patterns in production without hardening.
>
> - **Argo CD** — The server is patched to run with **`--insecure`**: the UI and API are meant to be used over **plain HTTP** (e.g. via ingress or port-forward). Session traffic and credentials are **not** protected like a TLS-terminated production install.
> - **metrics-server** — Deployed with **`--kubelet-insecure-tls`**, so it **does not verify** kubelet TLS certificates (common lab shortcut for kubeadm’s kubelet serving certs).
> - **Other lab choices** — Default **Argo CD admin password** in the Vagrantfile, **kubeconfig** on the shared `/vagrant` folder, **HTTP** listeners where noted (e.g. kagent UI on port 80), and **local-path** storage are all appropriate only for **local, disposable** clusters.

## Repository layout

| Path | Purpose |
|------|---------|
| `infrastructure/Vagrantfile` | Multi-node cluster (kubeadm), CNI, core addons, Argo CD bootstrap |
| `argocd/app-of-apps.yaml` | Root Argo CD `Application` that syncs everything under `argocd/apps/` |
| `argocd/apps/` | One `Application` manifest per component (sync waves order dependencies) |
| `argocd/templates/` | Raw Kubernetes YAML consumed by those apps (e.g. local-path-provisioner, Ollama/Gateway wiring) |
| `argocd/templates/kagent-mcps/` | kagent **`MCPServers`** CRs deployed into namespace `kagent` (kmcp controller) |
| `argocd/templates/kagent-agents/` | Custom kagent **`Agents`** CRs in namespace `kagent` |

## What Vagrant installs (VM provisioning)

The Vagrant environment brings up **one control-plane** (`k8s-master`, `192.168.56.10`) and **two workers** (`k8s-worker-1` … `192.168.56.11`, etc.) on **Ubuntu 24.04** (Bento box).

On **every** node:

- Swap disabled, kernel modules and sysctl for Kubernetes
- **containerd** (with systemd cgroup driver)
- **Kubernetes** packages pinned to **v1.35.x** (`kubelet`, `kubeadm`, `kubectl`)
- Kubelet bound to the **private network** interface (`eth1`)

On the **master** only:

- `kubeadm init` (API on `192.168.56.10`, pod CIDR `10.244.0.0/16`)
- **Cilium** CNI (CLI + cluster install) with kube-proxy replacement, **Ingress** enabled, **L2 announcements** and **LB IPAM**, pool **`192.168.56.20/28`**
- **metrics-server** (with tolerations for single-CP scheduling and `--kubelet-insecure-tls` for lab certs)
- **Argo CD v3.3.4** in namespace `argocd`, **TLS disabled on the server** (`--insecure`) so HTTP termination can sit in front
- **Ingress** for Argo CD: host **`argocd.local`** → `argocd-server` (Cilium ingress class)
- **Admin password** set from the Vagrant constant (default `qwer1234` — change this for anything beyond local lab use)
- **`/vagrant/vagrant.yaml`**: kubeconfig with server `https://192.168.56.10:6443` and context **`vagrant`**
- **`/vagrant/join.sh`**: worker join command

Workers run the common script, then join via `join.sh` when it appears.

**Not** installed by Vagrant: application charts, Gateway API CRDs as an Argo app, Agentgateway, kagent, local-path-provisioner, etc. Those are **Argo CD applications** (see below).

## What Argo CD installs (GitOps)

The [app of apps](argocd/app-of-apps.yaml) points at this repo’s `argocd/apps` on **`main`** (`repoURL: https://github.com/k0rvih/course-aire`). Sync uses **automated** sync with prune and self-heal.

| Application (Argo CD name) | Wave | What it deploys |
|----------------------------|------|------------------|
| `gateway-api-crds` | -2 | [Gateway API](https://github.com/kubernetes-sigs/gateway-api) standard CRDs (v1.5.0) |
| `agentgateway-crds-helm` | -1 | Agentgateway CRDs (Helm chart from `cr.agentgateway.dev`) |
| `local-path-provisioner` | 0 | Local path `StorageClass` / provisioner (lab only; manifests in this repo) |
| `agentgateway-helm` | 0 | **Agentgateway** controller (Helm, experimental Gateway API features enabled) |
| `agentgateway-ollama` | 1 | **Gateway**, **HTTPRoute**, **AgentgatewayBackend**, and **Ollama** `Service` + **EndpointSlice** — traffic to Ollama is sent to **`192.168.56.1:11434`** (VirtualBox host; run Ollama on the host, not only inside a pod) |
| `kagent-crds` | 2 | kagent CRDs (Helm from `ghcr.io/kagent-dev/kagent/helm`) |
| `kagent` | 3 | **kagent** (default provider **Ollama** at `ollama.agentgateway-system.svc.cluster.local:11434`, model **llama3.2**; UI **LoadBalancer** on port 80) |
| `kagent-mcps-servers` | 3 | Extra **MCP servers** for kagent: manifests in [`argocd/templates/kagent-mcps/`](argocd/templates/kagent-mcps/) — currently [AWS Documentation MCP Server](https://awslabs.github.io/mcp/servers/aws-documentation-mcp-server) as `MCPServer` **`aws-documentation-mcp`** (`uvx`, official AWS docs tools) |
| `kagent-agents` | 3 | Custom **kagent agents**: manifests in [`argocd/templates/kagent-agents/`](argocd/templates/kagent-agents/) — **`aws-expert`** uses the AWS documentation MCP tools from **`aws-documentation-mcp`** |

## Prerequisites

- [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org/)
- Enough RAM/CPU for 3 VMs (defaults: **2 vCPU / 6 GiB** per node)
- Optional but expected for the Ollama app: **Ollama** listening on the host at **`192.168.56.1:11434`** (VirtualBox host-only gateway from the guest’s perspective)

## Provision the cluster (Vagrant)

From the **repository root** (adjust paths if your clone lives elsewhere):

**PowerShell or Bash**

```bash
cd infrastructure
vagrant up
```

- First run builds **master**, then **worker-1** and **worker-2** (workers wait for `join.sh`).
- Kubeconfig on the host (same folder as the Vagrantfile, synced mount): **`infrastructure/vagrant.yaml`**.

**PowerShell** (from repo root)

```powershell
$env:KUBECONFIG = (Resolve-Path .\infrastructure\vagrant.yaml).Path
kubectl config use-context vagrant
kubectl get nodes
```

**Bash** (from repo root)

```bash
export KUBECONFIG="$(pwd)/infrastructure/vagrant.yaml"
kubectl config use-context vagrant
kubectl get nodes
```

Default Argo CD UI login: **`admin`** / password from `ARGOCD_ADMIN_PASSWORD` in the Vagrantfile (default **`qwer1234`**).

### Reach the Argo CD UI

1. Resolve **`argocd.local`** to the **external IP** Cilium assigns to the Argo CD Ingress (from the LB pool `192.168.56.20/28`), for example:

   **PowerShell or Bash** (`kubectl` is the same)

   ```bash
   kubectl get ingress -n argocd argocd-server-ingress -o wide
   ```

   Add a hosts entry, for example `<that-ip> argocd.local`:

   - **Windows:** `C:\Windows\System32\drivers\etc\hosts` (edit as Administrator)
   - **Linux / macOS:** `/etc/hosts` (with `sudo`)

2. Open **http://argocd.local** (server runs with `--insecure` by design in this lab).

Alternatively, use port-forward if you prefer not to use the ingress:

**PowerShell or Bash**

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Then browse **http://localhost:8080**.

## Provision workloads (Argo CD)

1. Ensure the cluster is up and you can `kubectl` with `vagrant.yaml`.

2. Register the app of apps (one-time, or after you change it):

   **Bash** (from repo root)

   ```bash
   kubectl apply -f ./argocd/app-of-apps.yaml
   ```

3. In the Argo CD UI (or CLI), confirm application **`aire-app-of-apps`** syncs; child apps under `argocd/apps/` should appear and sync in **wave** order.

**Forks / private clones:** [app-of-apps.yaml](argocd/app-of-apps.yaml) uses `repoURL: https://github.com/k0rvih/course-aire`. For your own fork, change `spec.source.repoURL` (and branch if needed) to match your Git remote before applying, or point Argo CD at your fork in the UI.

**kagent UI:** Exposed as `LoadBalancer`; pick up the address with:

**PowerShell or Bash**

```bash
kubectl get svc -n kagent
```

Addresses should come from the same Cilium pool as other LB services (`192.168.56.20/28`).

After **`kagent-agents`** syncs, the **aws-expert** agent appears in the kagent UI alongside the chart-bundled agents. The **AWS Documentation** MCP workload (`kagent-mcps-servers`) must be allowed **outbound HTTPS** so `uvx` can install the package and the server can call AWS documentation APIs.

## Teardown

From the **repository root**, enter **`infrastructure/`** (same as `vagrant up`):

**PowerShell or Bash**

```bash
cd infrastructure
vagrant destroy -f
```
