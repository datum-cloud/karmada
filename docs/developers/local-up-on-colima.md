# Running the local stack on Colima

`hack/local-up-karmada.sh` builds four kind clusters and a full control plane.
On macOS it assumes Docker Desktop. Under [Colima](https://github.com/abiosoft/colima)
several separate things break it, and most of them report a symptom several
layers away from the cause.

This page is what those failures look like and what fixes them, in the order you
meet them.

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

Run everything inside the VM. The Linux path has none of this.

```sh
colima ssh
git clone https://github.com/karmada-io/karmada.git ~/karmada
cd ~/karmada && ./hack/local-up-karmada.sh
```

The VM needs `go` matching `go.mod`, plus `kubectl`, `git` and `make`. The
script installs its own pinned `kind`. On an Ubuntu VM:

```sh
GO_VERSION=$(awk '$1 == "go" {print $2}' go.mod)
ARCH=$(dpkg --print-architecture)
curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" | sudo tar -C /usr/local -xz
sudo curl -fsSLo /usr/local/bin/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/${ARCH}/kubectl"
sudo chmod +x /usr/local/bin/kubectl
sudo apt-get install -y make
export PATH=/usr/local/go/bin:$PATH
```

## Keep kubeconfigs on the VM's own disk

Colima mounts your Mac home directory into the VM, so a checkout under it works
from both sides. The kubeconfigs cannot live there.

`kind` takes a lock by creating `<kubeconfig>.lock` with no permissions, which a
local filesystem allows its creator to open. The virtiofs mount refuses it, and
cluster creation fails after the nodes are already up:

```
ERROR: failed to create cluster: failed to lock config file: open .../karmada.config.lock: permission denied
```

The script then waits five minutes and reports only the consequence:

```
[ERROR] Timeout waiting for file exist .../karmada.config
```

The default `KUBECONFIG_PATH`, `~/.kube` inside the VM, is on the VM's disk.
Leave it there, or point it at another VM-local directory.

## Check which default route works

Outbound connections from the VM can start timing out partway through a run,
from the daemon and from the VM shell, while the Mac reaches the same hosts
fine. The first sign is usually a `go install` in the script failing:

```
dial tcp <address>:443: i/o timeout
```

A VM started with `--network-address` has two default routes, one on `eth0`
and one on `col0`, and either can be the one that stops working. You do not need
`--network-address` when everything runs inside the VM, and
`--network-address=false` does not remove the interface from an existing VM.

Test each interface, then delete the default route of the one that fails:

```sh
ip route show default
curl -sS -o /dev/null -m 10 -w '%{http_code}\n' --interface eth0 https://proxy.golang.org
curl -sS -o /dev/null -m 10 -w '%{http_code}\n' --interface col0 https://proxy.golang.org
sudo ip route del default via <gateway of the failing interface>
```

The deletion holds until the VM restarts.

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
kube-proxy  0/1  CrashLoopBackOff
E run.go:72] "command failed" err="failed complete: too many open files"
```

`kube-proxy` cannot start, so the service network is never programmed, so
`10.96.0.1` is unreachable, so CoreDNS cannot watch services, so the API server
cannot resolve etcd. One cause, four symptoms, and the useful one is last.

Raise the limits, but never lower one the VM already sets higher:

```sh
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances
sudo tee /etc/sysctl.d/99-kind.conf <<'CONF'
fs.inotify.max_user_watches=1048576
fs.inotify.max_user_instances=1024
CONF
sudo sysctl -p /etc/sysctl.d/99-kind.conf
```

Existing clusters recover once their `kube-proxy` pods restart. Delete the pods
in each cluster, and they come back healthy within a minute:

```sh
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-host -n kube-system delete pod -l k8s-app=kube-proxy
for c in member1 member2 member3; do
  kubectl --kubeconfig ~/.kube/members.config --context "$c" -n kube-system delete pod -l k8s-app=kube-proxy
done
```

A control plane that already failed to install needs the script run again.

## Rerun the whole script after a failure

A loaded VM can also lose a race in the routing step. The script reads each
member node's pod CIDR before the controller has assigned it and runs `ip route`
with an empty subnet, which prints the command's usage text and stops:

```
exec cmd in docker member1-control-plane ip route add  via 172.18.0.3
Usage: ip route { list | flush } SELECTOR
```

The CIDR is assigned seconds later, so running the script again succeeds.

Run it from the top rather than resuming partway. It deletes and recreates every
cluster first, so a full run is clean. Resuming after the kubeconfig merge is
not: with the temporary member kubeconfigs already removed, the merge writes an
empty `members.config` over the good one, and the only sign is

```
rm: missing operand
```

## Build only what you change

By default the script builds every component image from source. To test a
change to one component, bring the stack up from the published images and swap
in that one image, which keeps the build time and the disk to one component.

```sh
BUILD_FROM_SOURCE=false ./hack/local-up-karmada.sh
export KUBECONFIG=~/.kube/karmada.config
make image-karmada-controller-manager GOOS=linux GOARCH=$(go env GOARCH) VERSION=dev
kind load docker-image docker.io/karmada/karmada-controller-manager:dev --name karmada-host
kubectl --context karmada-host -n karmada-system set image \
  deployment/karmada-controller-manager karmada-controller-manager=docker.io/karmada/karmada-controller-manager:dev
kubectl --context karmada-host -n karmada-system scale deployment/karmada-controller-manager --replicas=1
```

With one replica, `kubectl logs deploy/karmada-controller-manager` reads the
leader. With two, it can pick the standby, which logs almost nothing. Check that
the pod running the new image holds the lease before reading its logs:

```sh
kubectl --context karmada-apiserver -n karmada-system get lease karmada-controller-manager \
  -o jsonpath='{.spec.holderIdentity}'
```

## Sizing

A VM with 8 CPUs and 16 GiB runs four kind clusters and the control plane with
about 11 GiB of memory still available. Memory is unlikely to be your problem, so
check `kube-proxy` before adding any.

Built from the published images plus one local image, the stack used 7.2 GiB of
Docker storage and 3.5 GiB on the VM's root disk, so `colima start --disk 12` is
enough. Colima's disk images grow on the Mac's disk up to their size limit, so
the limit is also how much of the Mac's free space the stack can take.
