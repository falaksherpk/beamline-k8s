# beamline-k8s

Kubernetes manifests for the beamline lab cluster (`k8cp`, `k8w1`, `k8w2`;
Kubernetes 1.36, built with kubeadm in Part 2 Chapter 13). Hosted on GitLab
(`beamline/beamline-k8s`, `origin`) with GitHub (`falaksherpk/beamline-k8s`,
`github`) as the disaster-recovery copy.

Node preparation (containerd, kubeadm/kubelet/kubectl, firewall, the kubeadm
config) lives in `beamline-ansible`, role `k8s`. This repository holds what is
applied to the cluster itself.

## Layout

- `bootstrap/` — applied by hand, before anything can manage the cluster
  from Git (the pod network has to exist first).
  - `calico/v3.32.2/` — Calico CNI via the Tigera Operator.
  - `argocd/v3.5.4/` — Argo CD, upstream non-HA `install.yaml` (unchanged).
- `argocd/` — Argo CD's own AppProjects and Applications (app of apps,
  managed by the `root` Application).
- `apps/` — application manifests that Argo CD deploys.

## Calico v3.32.2 bootstrap (as run in Chapter 13)

Upstream files, committed unchanged from
`https://raw.githubusercontent.com/projectcalico/calico/v3.32.2/manifests/`:

- `v1_crd_projectcalico_org.yaml` — CRDs
- `tigera-operator.yaml` — Tigera Operator (`quay.io/tigera/operator:v1.42.6`)

Verify them with `sha256sum -c SHA256SUMS` in that directory.

Our own resources:

- `installation.yaml` — VXLAN-only (`bgp: Disabled`), one IPv4 pool
  `10.244.0.0/16` (matches kubeadm's `podSubnet`), `encapsulation: VXLAN`,
  `natOutgoing: Enabled`, `blockSize: 26`; node address detected by CIDR
  `10.10.10.0/24` (each node also has `nat0` on 192.168.122.0/24).
- `apiserver.yaml` — Calico API server. Serves `projectcalico.org/v3`; without
  it, `tigerastatus/tiers` stays degraded (the `calico-system` tier is not
  created). Kubernetes NetworkPolicy was enforced either way (tested).

Order, on `k8cp` with a cluster-admin kubeconfig:

```bash
kubectl create -f v1_crd_projectcalico_org.yaml -f tigera-operator.yaml
kubectl -n tigera-operator rollout status deployment/tigera-operator
kubectl create -f installation.yaml
kubectl wait --for=create tigerastatus/calico --timeout=120s
kubectl wait --for=condition=Available tigerastatus/calico --timeout=300s
kubectl create -f apiserver.yaml
kubectl wait --for=create tigerastatus/apiserver --timeout=120s
kubectl wait --for=condition=Available tigerastatus/apiserver tigerastatus/tiers --timeout=300s
```

`kubectl create`, not `apply`: Calico's docs note the CRD bundle is too large
for client-side `apply`. `--for=create` first, because the operator creates
each `tigerastatus` object asynchronously.

## Argo CD v3.5.4 bootstrap (as run in Chapter 14)

Upstream file, committed unchanged from
`https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.4/manifests/install.yaml`
(non-HA; images `quay.io/argoproj/argocd:v3.5.4`, `ghcr.io/dexidp/dex:v2.45.1`,
`public.ecr.aws/docker/library/redis:8.2.3-alpine`). Verify it with
`sha256sum -c SHA256SUMS` in `bootstrap/argocd/v3.5.4/`.

Our own resources in that directory:

- `argocd-tls-certs-cm.yaml` — the lab CA under key `gitlab.beamline`, so
  repo-server trusts `https://gitlab.beamline`. Upstream ships this ConfigMap
  without data, so a server-side re-apply of `install.yaml` leaves our key
  alone (tested with a dry run).
- `argocd-server-nodeport.yaml` — UI/API on NodePort 30443 (443 -> 8080).
  Served on lab addresses only because kube-proxy has
  `nodePortAddresses: [10.10.10.0/24]` (`beamline-ansible`, role `k8s`);
  ufw cannot filter NodePorts.

Order, on `k8cp` with a cluster-admin kubeconfig:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f install.yaml
kubectl apply --server-side --field-manager=beamline-lab -f argocd-tls-certs-cm.yaml
kubectl apply --server-side --field-manager=beamline-lab -f argocd-server-nodeport.yaml
```

Never committed — each value comes from a hidden prompt (`read -rs`) or a file
and is piped into `kubectl`, never typed on a command line:

- Secret `repo-beamline-k8s` (label `argocd.argoproj.io/secret-type=repository`):
  `type=git`, `url=https://gitlab.beamline/beamline/beamline-k8s.git`,
  `username=gitlab+deploy-token-3`, `password` = GitLab project deploy token
  `argocd-beamline-k8s` (`read_repository` only, expires 2027-10-05).
- Secret `argocd-server-tls` (`kubernetes.io/tls`): `CN=argocd-server`, SANs
  `k8cp`/`k8w1`/`k8w2.beamline`, issued by the lab CA in
  `admin.beamline:~/lab-ca` (expires 2028-10-06).
- `argocd-secret` keys `admin.password` (bcrypt) and `admin.passwordMtime`;
  then `argocd-initial-admin-secret` deleted.
- `argocd-secret` key `webhook.gitlab.secret` = the secret token of the
  `beamline-k8s` project webhook (`https://k8cp.beamline:30443/api/webhook`,
  push events only, SSL verification on).

Outside this repository the webhook also needs GitLab to trust the lab CA
(`beamline-ansible`, role `gitlab`, tag `gitlab_trusted_certs`) and
`k8cp.beamline` in GitLab's outbound-request allowlist (Admin area ->
Settings -> Network).

Last, the only manual apply of the app-of-apps tree:

```bash
kubectl apply --server-side --field-manager=beamline-lab -f argocd/projects/platform.yaml
kubectl apply --server-side --field-manager=beamline-lab -f argocd/applications/root.yaml
```

`root` (project `platform`) then manages everything under `argocd/`, itself
included. `apps/beamline-nginx/` is the demo application (project `lab`,
namespace `beamline-demo`, Pod Security `restricted`).

UI: `https://k8cp.beamline:30443` (any node name works), user `admin`.
