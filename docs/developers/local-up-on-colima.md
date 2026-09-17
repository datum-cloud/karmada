# Running the local stack on Colima

`hack/local-up-karmada.sh` builds four kind clusters and a full control plane.
On macOS it assumes Docker Desktop. Under [Colima](https://github.com/abiosoft/colima)
four separate things break it, and three of them report a symptom several layers
away from the cause.

This page is what those failures look like and what fixes them.

## Run the script inside the VM

The script's macOS branch detects the Mac's network address and binds each
cluster's published ports to it. Docker Desktop shares the host network
namespace, so that works. Colima runs the daemon inside a VM that has no such
address, so it cannot.

The address has to be bindable in two places at once. `kind` runs on the Mac and
probes for a free port by binding the address there, while `docker` runs in the
VM and publishes on the same address. The Mac's address fails in the VM, the
VM's address fails on the Mac.

`127.0.0.1` satisfies both, through Colima's port forwarding, and then breaks
something else: every member cluster's kubeconfig says loopback, and the control
plane components that read it run inside the host cluster, where loopback means
themselves. The published address exists so those components can reach member
API servers, so collapsing it to loopback defeats the purpose.

Clone into the VM and run there. The Linux path has none of this.

```sh
colima ssh
git clone https://github.com/karmada-io/karmada.git ~/karmada
cd ~/karmada && ./hack/local-up-karmada.sh
```

The VM needs `go` matching `go.mod`, plus `kubectl`, `git` and `make`. The
script installs its own pinned `kind`.

## Leave `--network-address` alone

`colima start --network-address` gives the VM a routable address, which looks
like the answer to the section above. It is not, because `kind` still runs on
the Mac.

It also adds a second default route on a `col0` interface, after which outbound
connections start timing out, from the daemon and intermittently from the VM
shell. `--network-address=false` does not remove the interface on an existing
VM. Deleting the route by hand holds for a session:

```sh
colima ssh -- sudo ip route del default via 192.168.64.1 dev col0
```

## Give the daemon a Docker Hub mirror

Base image pulls may fail with `dial tcp <address>:443: i/o timeout`, a
different address each time, while the VM shell reaches the same registry
happily.

Check another registry before blaming the network. A `not found` from
`docker pull ghcr.io/...` proves the daemon's egress works and narrows it to
Docker Hub.

Add a mirror in the VM, keeping the keys Colima already set:

```json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "features": { "buildkit": true, "containerd-snapshotter": true },
  "registry-mirrors": ["https://mirror.gcr.io"]
}
```

Write that to `/etc/docker/daemon.json` and `systemctl restart docker`.

## Raise the inotify limits before creating clusters

Four clusters exceed the default limits, and the resulting failure points
somewhere else entirely.

What you see first is the control plane never coming up:

```
error creating storage factory: context deadline exceeded
lookup etcd-client.karmada-system.svc.cluster.local: i/o timeout
```

That reads as a DNS problem, and CoreDNS agrees:

```
[INFO] plugin/ready: Plugins not ready: "kubernetes"
```

Neither is the cause. Read down one more layer:

```
kube-proxy  0/1  Error
E run.go:72] "command failed" err="failed complete: too many open files"
```

`kube-proxy` cannot start, so the service network is never programmed, so
`10.96.0.1` is unreachable, so CoreDNS cannot watch services, so the API server
cannot resolve etcd. One cause, four symptoms, and the useful one is last.

```sh
sudo tee /etc/sysctl.d/99-kind.conf <<'CONF'
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=1024
CONF
sudo sysctl -p /etc/sysctl.d/99-kind.conf
```

Clusters created before the change do not recover. Delete them and run the
script again.

## Sizing

Four kind clusters and the control plane run comfortably in 8 CPU and 20 GiB,
with roughly 16 GiB still available and no swap in use. Memory is unlikely to be
your problem, so check `kube-proxy` before adding any.
