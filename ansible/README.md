# Ansible: the K3s cluster

These playbooks turn Ubuntu VMs on the LAN into one K3s cluster, and add or
remove VMs later. Each VM is either a **server** (control plane, with an etcd
member on it) or an **agent** (a worker).

With three servers the cluster survives losing any one of them: etcd still has
a majority, kubectl reaches one of the other two, and the playbooks keep
working. No server is special once the cluster exists.

## What you need

- **Ansible** on a Linux or macOS machine, or WSL on Windows (Ansible doesn't
  run on Windows itself), plus the collections in `requirements.yml`:

    ```bash
    ansible-galaxy collection install -r requirements.yml
    ```

- **Ubuntu VMs** (ours run 26.04; tested in Docker on 24.04) on one LAN that
  you can reach by SSH as `administrator` with your key, and where
  `administrator` can use sudo without a password: Ubuntu 26.04's sudo-rs
  prompts in a way Ansible doesn't recognise. Each VM needs internet access to
  download K3s.

## Set up the cluster

```bash
cd ansible
ansible-playbook site.yml
```

The VMs are listed in `inventory/hosts.yml`: three servers
(`ads-server-a` to `-c`) and three agents (`ads-agent-1` to `-3`) on
10.108.152.0/21. Each VM's `ansible_host` must be its LAN address, because the
other VMs use it to reach the cluster. Names become Kubernetes node names, so
they may only have lowercase letters, digits and `-`. Use 1, 3 or 5 servers;
the playbook refuses an even number, and it also refuses invalid names.

The playbook:

1. installs curl and the K3s installer on every VM, and opens the firewall
   if ufw is already on (see [Firewall](#firewall));
2. sets up the servers **one at a time**, each waiting until it is `Ready`
   before the next one starts. A new server joins through the first other
   server that answers on port 6443. Only when none answers does
   `ads-server-a`, the first in the inventory, start a new cluster
   (`cluster-init`); any other server stops with an error instead, so a
   second cluster is never started by accident;
3. joins the agents the same way, and waits until each one is `Ready`;
4. saves an admin kubeconfig as `ansible/kubeconfig` (git-ignored), taken from
   the first server it can reach and pointed at `k3s.ads.lan` (see
   [kubectl](#kubectl)).

The join token is read from a running server whenever it's needed, so it is
never stored in Git. Each node is named after its inventory name.

Running the playbook again is safe: nodes that are already in the cluster keep
the settings they joined with, and the run reports no changes on them.

## kubectl

The kubeconfig points at one name, `k3s.ads.lan`, that must resolve to **all
three servers**. kubectl tries the addresses in turn, so when a server is down
it moves on to the next one, with no virtual IP and nothing to switch by hand.
The API certificate accepts that name.

Add these lines to `/etc/hosts` on the machine you run kubectl from (in WSL,
the WSL distribution's `/etc/hosts`), once:

```text
10.108.159.43 k3s.ads.lan
10.108.153.26 k3s.ads.lan
10.108.159.83 k3s.ads.lan
```

Then:

```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes
```

The name can be changed with `k3s_api_name`. A DNS name with the three
addresses as A records works the same way.

## Add a VM

Add it to `k3s_servers` or `k3s_agents` in `inventory/hosts.yml` through a pull
request, then:

```bash
ansible-playbook site.yml --limit ads-agent-4
```

Any one server being down doesn't matter; the VM joins through another. Keep
the number of servers odd.

## Remove a VM

```bash
ansible-playbook remove.yml --limit ads-agent-3
```

This deletes the node from the cluster through another server that is up
(which also takes a server out of etcd), then uninstalls K3s from the VM.
Without `--limit` the playbook stops, unless you pass `-e remove_all=true` to
uninstall K3s from every VM.

To rebuild a VM, remove it, reinstall it, and run `site.yml --limit` for it: it
joins the existing cluster again, including `ads-server-a`.

## Upgrade K3s

Change `k3s_version` in `group_vars/all.yml` and run `site.yml`. Servers
upgrade one at a time; agents upgrade after them.

## Settings

All in `group_vars/all.yml`. Any of them can be set per VM in the inventory.

| Setting | Default | What it is |
|---|---|---|
| `k3s_version` | `v1.37.2+k3s1` | The K3s release |
| `k3s_api_name` | `k3s.ads.lan` | The name in the kubeconfig, resolving to every server |
| `k3s_node_ip` | the address on the VM's default route | The address K3s uses for the node |
| `k3s_extra_tls_sans` | none | Extra names for the API certificate |
| `k3s_lan_cidr` | `10.108.152.0/21` | The LAN, which ufw lets reach the cluster's own ports |
| `k3s_public_ports` | 80, 443, LiveKit 7881/7882/7891/7892 | Ports opened to everyone |

## Firewall

The playbook never turns ufw on, because that could lock you out of SSH. When
ufw is already active on a VM, it adds:

| Port | From | For |
|---|---|---|
| 6443/tcp (servers) | LAN | Kubernetes API |
| 2379–2380/tcp (servers) | LAN | etcd |
| 10250/tcp | LAN | kubelet |
| 8472/udp | LAN | flannel (pod network between nodes) |
| anything | 10.42.0.0/16, 10.43.0.0/16 | K3s's pod and service networks |
| 80, 443/tcp | anyone | Traefik |
| 7881, 7891/tcp; 7882, 7892/udp | anyone | LiveKit media for test and prod |

## How it was tested

On Docker containers running Ubuntu 24.04 with systemd (three servers, two
agents), with kubectl on a Linux machine whose hosts file maps `k3s.ads.lan`
to the three servers:

- the cluster came up with every node `Ready`, and a second run changed
  nothing on the nodes;
- with `ads-server-a` stopped, kubectl through `k3s.ads.lan` still listed and
  created resources, a new agent joined through `ads-server-b`, and rerunning
  the other nodes changed nothing;
- `ads-server-a` was removed and set up again, and it joined the existing
  cluster (its workloads were still there) instead of starting a new one;
- `remove.yml` took out a server and an agent, and `site.yml --limit` added
  them back;
- with ufw active, the rules above were added and the agent stayed `Ready`.

Not tested yet: the real VMs, and kubectl on Windows itself.
