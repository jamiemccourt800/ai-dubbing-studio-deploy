# Headlamp: the cluster dashboard

[Headlamp](https://headlamp.dev) shows what runs on the K3s cluster in a
browser: the workloads in every namespace, pod status and logs, events, nodes,
and CPU and memory use (from K3s's metrics-server). It replaces SSH and
`kubectl` for checking the cluster. Why Headlamp, and not the archived
Kubernetes Dashboard: ADR-0024 in the app repo (YGP-113).

| File | Holds |
|---|---|
| `headlamp.yaml` | The `headlamp` namespace, and a `HelmChart` that has K3s install the Headlamp chart (0.46.0) with a LAN-only Service on port 4466 |
| `viewer.yaml` | `headlamp-viewer`, the read-only login: Kubernetes' `view` role, plus reading nodes, namespaces, volumes and storage classes |

It is **not public**: there is no Ingress and no DNS record at Cloudflare, so
the Cloudflare Tunnel, which only goes to Traefik, never carries it. K3s's
built-in load balancer (ServiceLB) opens 4466 on every node, for the LAN and
for anyone on the Tailscale subnet router.

## Set up

Once per cluster:

```bash
kubectl apply -f cluster/headlamp/
kubectl -n headlamp rollout status deploy/headlamp   # "successfully rolled out"
kubectl -n headlamp get svc headlamp                 # EXTERNAL-IP lists the nodes
```

K3s's Helm controller installs the chart a few seconds after the apply, so
`rollout status` can say the Deployment isn't found at first; run it again.

## Open it

On the LAN, or on Tailscale with the subnet router, open
<http://k3s.ads.lan:4466>. `k3s.ads.lan` comes from your hosts file (see
`ansible/README.md`); any node's LAN address works too, for example
<http://10.108.159.43:4466>, since every node answers on 4466.

Headlamp asks for a token. Make one that lasts a day, and paste it into the
login page:

```bash
kubectl -n headlamp create token headlamp-viewer --duration=24h
```

Treat the token like a password: it's a login to the cluster. Don't paste it
into chats or tickets. It expires on its own; make a new one when it does.

## What the viewer can do

| What | Viewer token |
|---|---|
| Workloads, pods, logs and events in every namespace | Yes |
| Nodes, CPU and memory | Yes |
| Secrets | No |
| Change anything (restart, scale, delete, edit) | No |

Changes go through Git and Argo CD, as before. For a one-off change from the
UI, log in with a token that has more rights, such as one from your own
kubeconfig, and log out after.

Headlamp's own service account has no rights: the chart would bind it to
`cluster-admin`, but `headlamp.yaml` turns that off. Headlamp calls the
Kubernetes API with the token you log in with, so the token alone decides what
you can see and do.

## Things to know

- **Plain HTTP.** Through Tailscale the traffic is encrypted up to the subnet
  router, but it crosses the LAN unencrypted, token included. Accepted for the
  prototype, since the token is read-only and expires (ADR-0024).
- **Anyone on the LAN can open the login page.** Without a token they can't see
  anything.
- **Upgrading:** change `version` in `headlamp.yaml` and apply it again.
- **Removing it:** `kubectl delete -f cluster/headlamp/`. Deleting the
  `HelmChart` uninstalls the chart.
- **Without the LAN port,** it still works with
  `kubectl -n headlamp port-forward svc/headlamp 4466:4466` and
  <http://localhost:4466>.

Checked on 12 November 2026 on K3s v1.37.2 in Docker: the page loads on 4466
through ServiceLB; the viewer token lists pods in every namespace, nodes, logs
and node metrics, and gets `403` on Secrets and on deleting a pod; without a
token the API answers `401`.
