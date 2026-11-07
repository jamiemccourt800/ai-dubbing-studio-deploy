# Ansible: the K3s cluster

These playbooks turn Ubuntu VMs on the LAN into one K3s cluster, and add or
remove VMs later. Each VM is either a **server** (control plane, with an etcd
member on it) or an **agent** (a worker).

With three servers the cluster survives losing one: etcd still has a majority,
and the API answers from the other two. The first server in the inventory
starts the cluster, and the others join it.

## What you need

- **Ansible** on a Linux or macOS machine, or WSL on Windows (Ansible doesn't
  run on Windows itself), plus the collections in `requirements.yml`:

    ```bash
    ansible-galaxy collection install -r requirements.yml
    ```

- **Ubuntu VMs** (tested on 24.04) on one LAN that you can reach by SSH, as a
  user with sudo. Each one needs internet access to download K3s.

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
2. sets up the servers **one at a time**: the first one starts the cluster
   (`cluster-init`), and each of the others joins it and waits until it is
   `Ready` before the next one starts;
3. joins the agents and waits until each one is `Ready`;
4. saves an admin kubeconfig as `ansible/kubeconfig` (git-ignored), pointed at
   the first server:

    ```bash
    export KUBECONFIG=$PWD/kubeconfig
    kubectl get nodes
    ```

The join token is read from the first server whenever it's needed, so it is
never stored in Git. Each node is named after its inventory name.

Running the playbook again is safe: nodes that are already set up are left as
they are, and the run reports no changes.

## Add a VM

Add it to `k3s_servers` or `k3s_agents` in `inventory/hosts.yml` through a pull
request, then:

```bash
ansible-playbook site.yml --limit ads-agent-2
```

Keep the number of servers odd.

## Remove a VM

```bash
ansible-playbook remove.yml --limit ads-agent-2
```

This deletes the node from the cluster (which also takes a server out of
etcd), then uninstalls K3s from the VM. Without `--limit` the playbook stops,
unless you pass `-e remove_all=true` to uninstall K3s from every VM.

## Upgrade K3s

Change `k3s_version` in `group_vars/all.yml` and run `site.yml`. Servers
upgrade one at a time; agents upgrade after them.

## Settings

All in `group_vars/all.yml`. Any of them can be set per VM in the inventory.

| Setting | Default | What it is |
|---|---|---|
| `k3s_version` | `v1.37.2+k3s1` | The K3s release |
| `k3s_api_endpoint` | the first server's `ansible_host` | Where nodes join and kubectl connects |
| `k3s_node_ip` | the address on the VM's default route | The address K3s uses for the node |
| `k3s_extra_tls_sans` | none | Extra names for the API certificate, such as a DNS name |
| `k3s_lan_cidr` | `10.108.152.0/21` | The LAN, which ufw lets reach the cluster's own ports |
| `k3s_public_ports` | 80, 443, LiveKit 7881/7882/7891/7892 | Ports opened to everyone |

New VMs join through `k3s_api_endpoint`, so the first server must be up when
you add one. Nodes already in the cluster keep working without it: they know
every server.

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

On four Docker containers running Ubuntu 24.04 with systemd (three servers,
one agent):

- the cluster came up with all four nodes `Ready`, and a second run changed
  nothing;
- with the first server stopped, the API still accepted changes through
  another server;
- `remove.yml` took out a server and an agent, and `site.yml --limit` added
  them back;
- with ufw active, the rules above were added and the agent stayed `Ready`.

Not tested yet on the real VMs.
