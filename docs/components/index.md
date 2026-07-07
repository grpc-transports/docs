# Components

`grpc-transports` is a set of pure-Go modules (no cgo) that each carry gRPC over
a different secure carrier. They share one shape: a server constructor returning
a `net.Listener` and a client constructor returning a `grpc.DialOption`.

| Module | Import path | Carrier | What it does |
|--------|-------------|---------|--------------|
| [`wireguard`](wireguard.md) | `github.com/grpc-transports/wireguard` | WireGuard overlay | Inter-VM gRPC regardless of physical location. `ListenWireGuard` brings up a WireGuard device and serves overlay TCP connections; `DialOption` tunnels gRPC to an overlay address. Userspace (wireguard-go + gVisor netstack) or Linux-kernel backend. |
| [`ssh`](ssh.md) | `github.com/grpc-transports/ssh` | SSH tunnel | Transparent SSH-tunnelled gRPC. `ListenSSH` wraps inbound SSH connections as gRPC-ready `net.Conn`; `DialOption` opens gRPC channels over SSH. Ed25519 host keys, agent forwarding, pluggable `AuthCallback`. |
| [`vsock`](vsock.md) | `github.com/grpc-transports/vsock` | AF_VSOCK | Host↔guest gRPC over the Linux virtio-vsock family (KVM/QEMU or Apple VZ), addressed by `(context id, port)` with no IP stack in the guest. `Listen` returns a `net.Listener` whose connections report the peer CID; `Dialer.DialContext` matches `grpc.WithContextDialer`. Dependency-free, Linux-only with a portable non-Linux stub. |

`wireguard` and `ssh` depend on `google.golang.org/grpc` plus
`golang.org/x/crypto`; `wireguard` additionally pulls in `wireguard-go`, `wgctrl`,
`netlink`, and gVisor's netstack for the userspace data path. `vsock` has **no
third-party dependencies** — it is pure stdlib.
