# Podman Host Bridge Network Driver

A proof-of-concept Netavark plugin that allows **rootless** Podman containers to
use an existing host networking infrastructure based on Linux bridges, routing,
firewall rules and `dnsmasq`.

## Motivation

Rootless Podman normally uses `pasta` or `slirp4netns` for networking. While
convenient, these backends make it difficult to integrate containers into an
existing host networking setup and to enforce fine-grained firewall policies. In
particular, restricting container traffic to selected subnets while preventing
access to other locally reachable networks (such as VPNs) is not
straightforward.

This project allows Podman containers to reuse the same networking
infrastructure already used by virtual machines (libvirt/QEMU) and LXC
containers, including existing bridge interfaces, routing, firewall rules and
DHCP/DNS services.

## How it works

The Netavark plugin creates a request describing the desired network
configuration and sends it to a privileged helper daemon over a Unix domain
socket.

The request includes an open file descriptor referring to the target network
namespace, passed using `SCM_RIGHTS`.

This design is necessary because Netavark plugins execute inside the
container's user, network and mount namespaces. As a result:
- `sudo` cannot be used to perform privileged operations on the host.
- The namespace path visible to the plugin is generally not valid in the host
  mount namespace.
- Passing an open namespace file descriptor avoids both problems.

The privileged daemon receives the namespace file descriptor, creates the
required `veth` pair, moves one end into the target network namespace, and
connects the other end to the configured bridge.

The Netavark plugin has `CAP_NET_ADMIN` in the container's namespaces, where the
unprivileged user is mapped as root. Therefore, it has full control over the
container end of the veth pair, and the rest of the configuration (such as
setting the interface up and configuring mac/ip addresses) is done by the plugin.

## Status

This project is currently a **proof of concept**.

It is implemented in Python and invokes `iproute2` commands (`ip link`,
`ip addr`, `ip route`, etc.) to manipulate networking.

A production-quality implementation would likely:

- be written in a compiled language (C or Rust),
- communicate directly with the kernel using Netlink,
- avoid spawning external commands.

However, for the intended use case (starting a handful of containers
occasionally), the current implementation is more than enough.

## Security

The helper daemon communicates over a Unix domain socket.

At the moment the socket is created under `/tmp` because it is visible both to
the plugin and to the host. The socket has a dedicated `podman-netd` user and
group, and has mode 0660.
- Only members of the `podman-netd` group can open the socket for connecting.
  This prevents random users in the system from creating veth pairs.
- Even though the socket resides in `/tmp`, it cannot be hijacked because
  typically `/tmp` has the sticky bit set.

The privileged daemon uses a small configuration file, and the accepted bridge
interfaces are explicitly listed there. That means even users who are allowed to
connect to the socket cannot add veth pairs to just any bridge interface in the
system.

## Limitations

- Prototype-quality implementation.
- Linux only.
- Requires a privileged helper daemon running on the host.
- Assumes networking is managed externally (Linux bridges, routing, firewall rules, `dnsmasq`, etc.).
- No IP address management beyond what Podman/Netavark already provides.

## Installing

```
# Register local plugin path
mkdir -p ~/.config/containers
cp containers.conf ~/.config/containers/

# Set up local plugin path and copy plugin
mkdir -p ~/.local/share/containers/netavark
cp host-bridge ~/.local/share/containers/netavark

# Verify that podman can "see" the plugin
podman info --format {{.Plugins.Network}}

# Create a dedicated network using the plugin
podman network create --driver host-bridge --subnet 192.168.1.0/24 --gateway 192.168.1.1 host-virbr0
# Or a host-only variant with no gateway
podman network create --driver host-bridge --subnet 192.168.1.0/24 --opt=no_default_route host-virbr0

# Set up socket permissions
sudo useradd -r podman-netd -d /
sudo usermod -a myuser -G podman-netd

# Install the privileged daemon
sudo cp podman-netd /usr/local/libexec/
sudo cp systemd/podman-netd.service /etc/systemd/system/podman-netd.service
sudo cp systemd/podman-netd.socket /etc/systemd/system/podman-netd.socket
sudo cp podman-netd.conf /etc/
sudo systemctl daemon-reload
```

## Testing

```
# Start the privileged daemon
sudo systemctl start podman-netd.socket

# Run a simple test container
podman run --rm --network=host-virbr0:mac=02:00:12:34:56:78,ip=192.168.1.2,bridge=virbr0 --dns 192.168.1.1 --no-hosts -it alpine
```
