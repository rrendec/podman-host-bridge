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
podman network create --driver host-bridge host-bridge

# Install the privileged daemon
sudo cp podman-netd /usr/local/libexec/
```

## Testing

```
# Start the privileged daemon
sudo /usr/local/libexec/podman-netd

# Run a simple test container
podman run --rm --network=host-bridge -it alpine
```
