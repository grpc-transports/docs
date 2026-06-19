# grpc-transports

Pure-Go **custom gRPC transports**. Each module exposes the same two-sided API —
a `net.Listener` on the server (hand it to `grpc.Server.Serve`) and a
`grpc.DialOption` on the client (pass it to `grpc.Dial`) — so swapping the
underlying carrier is a one-line change to the rest of your service. No cgo.

| Transport | Carries gRPC over | Authentication |
|-----------|-------------------|----------------|
| [`wireguard`](components/wireguard.md) | a WireGuard overlay (userspace wireguard-go + gVisor netstack, or the Linux kernel module) | Curve25519 peer keys |
| [`ssh`](components/ssh.md) | an SSH tunnel | Ed25519 host keys + `authorized_keys` / agent keys, with a pluggable `AuthCallback` |

Both grew out of [weft](https://github.com/openweft/weft)'s need to reach a gRPC
agent from a controller: `ssh` fits a human-driven CLI client with per-user
keys, while `wireguard` fits inter-VM, controller-provisioned mesh traffic where
SSH's per-user model is a poor fit.

## Components

| Module | Import path | What it does |
|--------|-------------|--------------|
| [`wireguard`](components/wireguard.md) | `github.com/grpc-transports/wireguard` | gRPC over a WireGuard overlay for inter-VM communication regardless of physical location. Two backends ship side by side: a userspace data path (wireguard-go + gVisor netstack, no privileges, any OS) and a Linux kernel backend (`CAP_NET_ADMIN`, line-rate). |
| [`ssh`](components/ssh.md) | `github.com/grpc-transports/ssh` | gRPC over a transparent SSH tunnel. Ed25519 host-key auto-generation, SSH agent forwarding for client auth, and a pluggable `AuthCallback` seam for verifiers like OpenPubkey. |

## How it fits together

```
grpc.Server.Serve(  Listen{WireGuard,SSH}(addr, cfg)  )   ← server side
grpc.Dial(target,   DialOption(addr, cfg)            )    ← client side
```

The server constructor returns a `net.Listener` whose accepted connections are
already carried over the chosen overlay/tunnel; the client constructor returns a
`grpc.DialOption` that dials through the same carrier. gRPC itself is unmodified
— it never sees the transport beneath it.

## Choosing a transport

- **Inter-VM gRPC across hosts / zones**, no existing overlay, controller can
  distribute keys → [`wireguard`](components/wireguard.md).
- **Human-driven CLI client** with per-user SSH keys / agent forwarding →
  [`ssh`](components/ssh.md).
- **VM ↔ VM on the same host** → prefer vsock (neither module is needed).
