# AI Dubbing Studio: deployment

The Kubernetes manifests for [AI Dubbing Studio](https://github.com/WH1TE-SHAD0W/ai-dubbing-studio).
Argo CD watches this repository and makes the K3s cluster match it, so nobody
deploys by hand ([docs](https://wh1te-shad0w.github.io/ai-dubbing-studio/deployment/)).

| Repository | Holds |
|---|---|
| [`ai-dubbing-studio`](https://github.com/WH1TE-SHAD0W/ai-dubbing-studio) | The code. Its Images workflow builds the images and pushes them to Docker Hub. |
| `ai-dubbing-studio-deploy` (this one) | What runs where: which images, with which settings. |

The manifests are kept apart from the code, as Argo CD recommends: changing
what runs never triggers the app's CI, and this repository's history is a log of
releases. **No secrets are kept here**; they're created on the cluster by hand
(see [Secrets](#secrets)).

## Layout

```text
base/                  what every environment runs
  api.yaml             the Hono API with the built app, its Service and Ingress
  postgres.yaml        PostgreSQL 18 with a 2 GB volume
  livekit.yaml         LiveKit, its config, and its media ports on the node
overlays/
  test/                namespace ads-test, images tagged "testing"
  prod/                namespace ads-prod, images tagged "prod", media ports 7891/7892
  local/               namespace ads-local, an image you built, for trying it with k3d
cluster/
  traefik-config.yaml  once per cluster: keep visitors' real addresses (see below)
  cloudflared.yaml     once per cluster: the Cloudflare Tunnel, the public endpoint
```

| Overlay | Namespace | API image | Host name | LiveKit media ports |
|---|---|---|---|---|
| test | `ads-test` | `glossia/ai-dubbing-studio-api:testing` | `test.glossia.ie` | 7881 TCP, 7882 UDP |
| prod | `ads-prod` | `glossia/ai-dubbing-studio-api:prod` | `glossia.ie` | 7891 TCP, 7892 UDP |
| local | `ads-local` | `ai-dubbing-studio-api:local` | `ads.localhost` | 7881 TCP, 7882 UDP |

Each overlay sets its host in `BETTER_AUTH_URL` and in its Ingresses. Test and
prod are public through the Cloudflare Tunnel (see
[Public endpoint](#public-endpoint-cloudflare-tunnel)).

Not here yet: Redis and the translation worker pool (ADR-0018). They come with
the translator work in the app repository.

To see exactly what an overlay creates, without a cluster:

```bash
kubectl kustomize overlays/test
```

Every pull request is checked by the **Validate** workflow: each overlay must
render, and what it renders must be valid Kubernetes
([kubeconform](https://github.com/yannh/kubeconform)). The README is linted too.

`main` is protected (`.github/ruleset-main.json`, active on GitHub): every
change is a pull request with one approval, both checks must pass, and `main`
can't be deleted or force-pushed. Nobody can skip these rules.

## Set up a cluster

Once per cluster, before the first deploy:

1. **Keep visitors' real addresses.** Otherwise the API would put every
   visitor in one login rate-limit bucket. `cluster/traefik-config.yaml` tells
   K3s's Traefik to keep them: `externalTrafficPolicy: Local` for requests from
   the LAN, which K3s's load balancer would otherwise change to `10.42.0.1`, and
   trusting `X-Forwarded-For` from the pod network for requests through the
   tunnel:

    ```bash
    kubectl apply -f cluster/traefik-config.yaml
    ```

2. **Create each environment's Secrets** (next section).
3. **Open the public endpoint** (see
   [Public endpoint](#public-endpoint-cloudflare-tunnel)).

## Secrets

Three Secrets per namespace. Use your own random values, for example from
`openssl rand -hex 24`, and different ones for test and prod:

```bash
NS=ads-test   # or ads-prod
kubectl create namespace $NS
kubectl -n $NS create secret generic postgres --from-literal=password=<random>
kubectl -n $NS create secret generic livekit  --from-literal=api-key=<name> --from-literal=api-secret=<at least 32 random characters>
kubectl -n $NS create secret generic api      --from-literal=better-auth-secret=<at least 32 random characters>
```

| Secret | Keys | Used by |
|---|---|---|
| `postgres` | `password` | Postgres, and the API's `DATABASE_URL` |
| `livekit` | `api-key`, `api-secret` | LiveKit, and the API (rooms and tokens) |
| `api` | `better-auth-secret` | The API's login cookies. Changing it logs every host out. |

## Public endpoint (Cloudflare Tunnel)

`cluster/cloudflared.yaml` runs two cloudflared pods, on different nodes, that
keep a tunnel open from the cluster to Cloudflare. Visitors reach `glossia.ie`
and `test.glossia.ie` at Cloudflare. Cloudflare serves HTTPS with its own
certificate and renews it, and the tunnel carries every request (WebSockets
too) to Traefik, which sends each host name to its environment. No port is
opened on the VMs or the router for this.

Set it up once, from a machine with kubectl access (the relay):

```bash
# cloudflared, from Cloudflare's apt repository
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared

cloudflared tunnel login                 # opens a link: choose glossia.ie
cloudflared tunnel create glossia
cloudflared tunnel route dns glossia glossia.ie
cloudflared tunnel route dns glossia test.glossia.ie

# The tunnel's token is the only secret: it goes in the cluster, not in Git
kubectl create namespace cloudflared
kubectl -n cloudflared create secret generic tunnel --from-literal=token="$(cloudflared tunnel token glossia)"
kubectl apply -f cluster/cloudflared.yaml
kubectl -n cloudflared get pods          # both Running and 1/1
```

`route dns` creates a CNAME record for each name, pointing at the tunnel. It
fails if the name already has a DNS record; delete that record first. Create the
tunnel with the CLI as above, not in the Cloudflare dashboard. A dashboard
tunnel ignores the routes in `cluster/cloudflared.yaml` and takes them from the
dashboard instead.

In the Cloudflare dashboard, under **SSL/TLS → Edge Certificates**, turn on
**Always Use HTTPS**, so plain HTTP is redirected to HTTPS.

Cloudflare renews its certificate itself; there is nothing to renew by hand.

### Check it works

Once an environment is deployed:

```bash
# HTTPS with a valid certificate: {"status":"ok"}
curl https://test.glossia.ie/ready

# Plain HTTP is redirected: 301 to https://
curl -sI http://test.glossia.ie | head -n 3

# /internal is refused: 403 (blocked at the Ingress, YGP-71)
curl -s -o /dev/null -w '%{http_code}\n' https://test.glossia.ie/internal/x
```

The API has no WebSocket of its own yet, so check WebSockets with a throwaway
echo server on the test host, then remove it:

```bash
kubectl -n ads-test create deployment ws-echo --image=jmalloc/echo-server:v0.3.7 --port=8080
kubectl -n ads-test expose deployment ws-echo --port=80 --target-port=8080
kubectl -n ads-test create ingress ws-echo --rule="test.glossia.ie/ws-echo=ws-echo:80"

# Expect "HTTP/1.1 101 Switching Protocols"
curl -si --http1.1 --max-time 5 https://test.glossia.ie/ws-echo \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" | head -n 1

kubectl -n ads-test delete ingress,service,deployment ws-echo
```

A new environment needs only its DNS record (`cloudflared tunnel route dns
glossia <host>`) and its host name in its overlay. The tunnel sends every name
to Traefik.

LiveKit's media (UDP and TCP on 7881/7882 and 7891/7892) can't go through the
tunnel, which carries only HTTP. It's set up with LiveKit (YGP-68).

## Things to know

- **Automatic service variables are off** (`enableServiceLinks: false`) in every
  pod. Kubernetes would otherwise add `<SERVICE>_PORT` variables, and a Service
  called `livekit` sets `LIVEKIT_PORT`, which LiveKit reads as its own port and
  crashes.
- **The API trusts K3s's pod network** (`TRUSTED_PROXIES=10.42.0.0/16`), where
  Traefik runs, so it believes Traefik's `X-Forwarded-For`.
- **LiveKit's media ports are opened on the node** (`hostPort`), and LiveKit
  tells browsers the node's own address. That's why test and prod use different
  ports, and why LiveKit restarts with `Recreate` (one pod at a time).
- **The API restarts a few times on a fresh deploy** until Postgres is ready.
  That's expected: it exits if it can't reach the database, and then stays up.

## Try it on your own machine (k3d)

[k3d](https://k3d.io) runs K3s inside Docker, so you can try the manifests
without the server. Nothing is reachable from outside your machine except the
cluster's API port, which k3d publishes on all addresses.

```bash
# A one-node cluster, with Traefik on http://localhost:8080
k3d cluster create ads -p "127.0.0.1:8080:80@loadbalancer"

# The API image, built from the app repository and loaded into the cluster
docker build -f apps/api/Dockerfile -t ai-dubbing-studio-api:local .   # in the app repo
k3d image import ai-dubbing-studio-api:local -c ads

kubectl apply -f cluster/traefik-config.yaml
kubectl apply -k overlays/local
# create the three Secrets in ads-local, as above
kubectl -n ads-local get pods        # wait until all are Running and 1/1

curl -H "Host: ads.localhost" http://localhost:8080/ready   # {"status":"ok"}
```

Open the app by sending `Host: ads.localhost` to `localhost:8080`, or add
`127.0.0.1 ads.localhost` to your hosts file and open <http://ads.localhost:8080>.

Two problems you may hit on Windows with Docker Desktop:

- **K3s stops with "kubelet is configured to not run on a host using cgroup
  v1"**: Docker is using the older cgroup v1. Create the cluster with
  `--k3s-arg "--kubelet-arg=fail-cgroupv1=false@server:*"`. (The Ubuntu server
  uses cgroup v2 and doesn't need it.)
- **`kubectl` times out connecting to `host.docker.internal`**: point it at your
  own machine instead:
  `kubectl config set-cluster k3d-ads --server=https://127.0.0.1:<port>`, with
  the port from `docker port k3d-ads-serverlb 6443`.

When you're done: `k3d cluster delete ads`.
