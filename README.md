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
  api.yaml             the Hono API with the built app, its Service and Ingress,
                       and the Ingress that blocks /internal
  postgres.yaml        PostgreSQL 18 with a 2 GB volume
  redis.yaml           Redis 8 for the translation jobs, with a 1 GB volume
  livekit.yaml         LiveKit, its config, and its media ports on the node
  translator.yaml      the translation workers: a warm pool of 2
overlays/
  test/                namespace ads-test, images tagged "testing"
  prod/                namespace ads-prod, images tagged "prod", media ports 7891/7892
  local/               namespace ads-local, an image you built, for trying it with k3d
cluster/
  traefik-config.yaml  once per cluster: keep visitors' real addresses (see below)
  cloudflared.yaml     once per cluster: the Cloudflare Tunnel, the public endpoint
  argocd-lan.yaml      once per cluster: the Argo CD UI on the LAN, port 8443
argocd/
  test.yaml            the Argo CD application that keeps ads-test in sync
  prod.yaml            the same for ads-prod (not applied yet)
ansible/               builds the K3s cluster itself from the VMs (see ansible/README.md)
```

| Overlay | Namespace | Image tag | Host name | LiveKit media ports |
|---|---|---|---|---|
| test | `ads-test` | `testing` | `ads-test.localhost` | 7881 TCP, 7882 UDP |
| prod | `ads-prod` | `prod` | `ads.localhost` | 7891 TCP, 7892 UDP |
| local | `ads-local` | `local` (built yourself) | `ads.localhost` | 7881 TCP, 7882 UDP |

The images are `glossia/ai-dubbing-studio-api` and
`glossia/ai-dubbing-studio-translator` on Docker Hub, built by the app repo's
Images workflow.

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

The cluster itself, three servers and three agents on the LAN, is built with
Ansible: see [ansible/README.md](ansible/README.md). It also saves the
kubeconfig the commands below use.

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

Five Secrets per namespace. Use your own random values, for example from
`openssl rand -hex 24`, and different ones for test and prod:

```bash
NS=ads-test   # or ads-prod
kubectl create namespace $NS
kubectl -n $NS create secret generic postgres --from-literal=password=<random>
kubectl -n $NS create secret generic livekit  --from-literal=api-key=<name> --from-literal=api-secret=<at least 32 random characters>
kubectl -n $NS create secret generic api      --from-literal=better-auth-secret=<at least 32 random characters>
kubectl -n $NS create secret generic internal --from-literal=token=<at least 32 random characters>
kubectl -n $NS create secret generic translator --from-literal=gemini-api-key=<your Gemini API key>
```

| Secret | Keys | Used by |
|---|---|---|
| `postgres` | `password` | Postgres, and the API's `DATABASE_URL` |
| `livekit` | `api-key`, `api-secret` | LiveKit, and the API (rooms and tokens) |
| `api` | `better-auth-secret` | The API's login cookies. Changing it logs every host out. |
| `internal` | `token` | The API and the workers, for the workers' `/internal` calls |
| `translator` | `gemini-api-key` | The workers' Gemini Live API key |

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
tunnel, which carries only HTTP. It comes in through the Hetzner relay
instead (ADR-0023 in the app repo, YGP-109), which isn't part of this setup.

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
- **The workers don't translate real audio yet.** Until they connect to
  LiveKit (YGP-70), they use their WAV adapter: they start, take jobs and
  claim tracks, but have no audio to work on.
- **`/internal` is blocked at the Ingress.** It's the translator workers' API,
  and they call the API's Service directly (`http://api`). Through Traefik,
  `/internal` and everything under it answer `403 Forbidden`: a second
  Ingress for that path uses the `deny-internal` middleware, which only allows
  `127.0.0.1`. Each overlay names the middleware with its namespace
  (`ads-test-deny-internal@kubernetescrd`). To check it on a cluster:
  `curl -i http://ads-test.localhost/internal/tracks/x/claim` should give `403`.

## Argo CD

Argo CD (v3.5.6) runs on the K3s cluster in the `argocd` namespace (YGP-82).
Each environment is one Argo CD application, in `argocd/`. It points at the
environment's overlay on `main` and syncs on its own:

- **A merge to `main`** that changes the overlay or `base/` reaches the cluster
  within about 3 minutes (Argo CD checks Git every 3 minutes).
- **prune**: a resource removed from Git is deleted from the cluster.
- **selfHeal**: a change made on the cluster by hand is undone.

The Secrets aren't in Git, so Argo CD doesn't track them and never deletes
them.

| Application | Overlay | Namespace | Status |
|---|---|---|---|
| `ads-test` | `overlays/test` | `ads-test` | Set up |
| `ads-prod` | `overlays/prod` | `ads-prod` | Written (`argocd/prod.yaml`), not applied: once something reaches the app repo's `main` and the `ads-prod` Secrets exist |

Set up once, after the Secrets exist:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.6/manifests/install.yaml
kubectl apply -f argocd/test.yaml
```

Check it:

```bash
kubectl -n argocd get applications     # SYNC STATUS Synced, HEALTH Healthy
```

### The Argo CD UI

The UI shows each application's sync and health, every resource it manages,
and the pods' logs. It's on the LAN at <https://k3s.ads.lan:8443> (user
`admin`), reached from the LAN or through the Tailscale subnet router, the same
way as the cluster dashboard (ADR-0024 in the app repo). It isn't public: there
is no Ingress or Cloudflare record for it. Argo CD serves HTTPS itself with a
self-signed certificate, so the browser warns the first time.

Set up once, after Argo CD is installed. **Change the admin password first**,
because the address is open to the whole LAN:

1. Forward the port, and open <https://localhost:8443> while it runs:

    ```bash
    kubectl -n argocd port-forward svc/argocd-server 8443:443
    ```

2. Log in as `admin` with the first password. Read it yourself in your own
   terminal, and never paste it into chats or tickets:

    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
    ```

3. In the UI, open **User Info → Update Password** and set a new one. Then stop
   the port-forward.
4. The first password is no longer needed:

    ```bash
    kubectl -n argocd delete secret argocd-initial-admin-secret
    ```

5. Open the UI on the LAN:

    ```bash
    kubectl apply -f cluster/argocd-lan.yaml
    kubectl -n argocd get svc argocd-server-lan   # EXTERNAL-IP lists the nodes
    ```

`k3s.ads.lan` comes from your hosts file (see `ansible/README.md`); any node's
LAN address works too, since every node answers on 8443. If ufw is active on
the nodes, 8443 needs a rule for the LAN (`k3s_lan_cidr`), like the cluster's
other own ports.

Without the LAN Service, the UI is still reachable with
`kubectl -n argocd port-forward svc/argocd-server 8443:443` and
<https://localhost:8443>.

**Not yet: new images.** The test overlay uses the moving `testing` tag, so
the manifests don't change when a new image is pushed, and Argo CD doesn't
notice it. Until that's decided (ADR-0021 in the app repo), restart the API to
pull the new image: `kubectl -n ads-test rollout restart deploy/api`.

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
