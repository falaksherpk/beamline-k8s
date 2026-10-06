# Cluster smoke tests

Throwaway checks used in Part 2 Chapter 13, kept so they can be re-run after
an upgrade or a rebuild. Run on `k8cp` with a cluster-admin kubeconfig.
Each test creates its own namespace; delete it afterwards.

## Network: cross-node pods, Service DNS, egress, NetworkPolicy

`nettest.yaml` puts `server` (busybox httpd on 8080, Service `nettest-server`
on port 80) on `k8w2` and `client` on `k8w1` (pinned by
`kubernetes.io/hostname`; adjust if the nodes are renamed), so every request
crosses VXLAN between the two workers.

```bash
kubectl apply -f nettest.yaml
kubectl wait --for=condition=Ready pod --all -n nettest --timeout=180s
SERVER_IP=$(kubectl get pod -n nettest server -o jsonpath='{.status.podIP}')
for u in "http://$SERVER_IP:8080" http://nettest-server.nettest.svc.cluster.local http://example.com; do
  kubectl exec -n nettest client -- wget -qO- -T 5 "$u" >/dev/null && echo "OK   $u" || echo "FAIL $u"
done
```

Pass: all three `OK`.

NetworkPolicy enforcement: `kubectl apply -f deny-server.yaml`, wait a few
seconds, repeat the first two requests — both must time out — then
`kubectl delete -f deny-server.yaml` and they must succeed again.

Clean up: `kubectl delete namespace nettest`.

## Registry: pull a private image from gitlab.beamline:5050

`pulltest.yaml` pulls `beamline/podman-lab/beamline-test:v1` with
`imagePullPolicy: Always`, so the pull must authenticate. It needs namespace
`regtest` and the pull Secret first. The token is read at a hidden prompt and
passed through a pipe, never on the command line:

```bash
kubectl create namespace regtest
read -rsp 'beamline-registry token: ' TOKEN && echo
auth=$(printf '%s:%s' 'gitlab+deploy-token-2' "$TOKEN" | base64 -w0)
printf '{"auths":{"gitlab.beamline:5050":{"auth":"%s"}}}' "$auth" \
  | kubectl create secret generic beamline-registry -n regtest \
      --type=kubernetes.io/dockerconfigjson --from-file=.dockerconfigjson=/dev/stdin
unset TOKEN auth
kubectl apply -f pulltest.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/pulltest -n regtest --timeout=120s
kubectl logs -n regtest pulltest
```

Pass: logs `pulled-ok` and `3.24.2`; the pod's `imageID` ends in
`sha256:78c09366dc0109386bdb0b6c55c6c164aed1b8987384e046eddd3fe8cf297364`.

Clean up — the Secret holds a write-capable token, so do not leave it behind:
`kubectl delete namespace regtest`, then remove the cached image from the node
that ran the pod (`sudo crictl rmi gitlab.beamline:5050/beamline/podman-lab/beamline-test:v1`).
