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
