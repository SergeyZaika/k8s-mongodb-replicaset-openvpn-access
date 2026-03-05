# Secure MongoDB ReplicaSet access for Kubernetes clusters via OpenVPN

Access to MongoDB ReplicaSets inside Kubernetes through OpenVPN, without exposing database ports to the internet.

This is an **architecture repository**. It documents the infrastructure access model, network topology, and Kubernetes implementation — not an OpenVPN installation tutorial.

---

## Table of Contents

- [Introduction](#introduction)
- [Problem](#problem)
- [Architecture](#architecture)
- [Why VPN](#why-vpn)
- [Why OpenVPN](#why-openvpn)
- [Kubernetes Implementation](#kubernetes-implementation)
- [Security Considerations](#security-considerations)
- [Repository Structure](#repository-structure)
- [External Dependencies](#external-dependencies)

---

## Introduction

MongoDB ReplicaSets running inside Kubernetes are not accessible from outside the cluster without exposing ports. Exposing port 27017 directly — even behind a load balancer — creates an unacceptable attack surface for a database tier.

This repository demonstrates a controlled access model: a purpose-scoped OpenVPN server runs inside Kubernetes, VPN clients connect through it, and NetworkPolicy restricts egress from the VPN pod strictly to MongoDB pods on port 27017.

This pattern is used when database ports must remain private while still allowing controlled operational access for developers, migration tooling, and administrative tasks.

### Architecture Scope

The primary goal of this repository is to demonstrate secure administrative access to MongoDB ReplicaSets inside Kubernetes.

The `data-access` directory contains the core architecture used for database access.

The `monitoring` directory provides an optional extension showing how the same VPN access pattern can be reused to access internal monitoring systems such as Prometheus.

The monitoring configuration is **not required** for MongoDB access and is included only as an additional example.

### Core Principle

Databases should never be exposed directly to the internet.

Administrative database access must pass through a controlled infrastructure access layer.

In this architecture, that layer is a dedicated OpenVPN gateway running inside Kubernetes.

---

## Problem

- MongoDB ReplicaSet members communicate using internal cluster DNS and pod IPs
- External clients (developers, ops tooling, migration scripts) need direct ReplicaSet access for administrative operations
- Exposing MongoDB ports to the internet is not acceptable
- The cluster runs across multiple namespaces (`dev`, `stage`, multiple `prod-*` environments)

---

## Architecture

```
Internet
   |
External TCP Load Balancer
   |
TCP :1194
   |
NodePort 31194
   |
Service: openvpn-db (namespace: vpn)
   |
OpenVPN Pod
   |
VPN network: 10.88.0.0/24
   |
Kubernetes pod network (10.233.0.0/18, 10.233.64.0/18)
   |
MongoDB ReplicaSet pods (:27017)
```

Any external **L4 TCP load balancer** can be used at the ingress point:

- AWS NLB
- GCP TCP Load Balancer
- Azure Load Balancer
- MetalLB
- Hetzner Load Balancer

The load balancer must support **TCP passthrough** — OpenVPN runs in TCP mode and the raw TCP stream must reach the NodePort without HTTP-level termination or inspection.

The OpenVPN server pushes routes for both Kubernetes pod CIDRs (`10.233.0.0/18`, `10.233.64.0/18`) and the cluster DNS (`10.233.0.3`), with search domains `cluster.local` and `svc.cluster.local`. This allows VPN clients to resolve MongoDB service names using internal cluster DNS.

---

## Why VPN

- **No port exposure**: MongoDB never has a publicly routable endpoint
- **Encrypted transport**: all traffic between client and cluster is encrypted in the VPN tunnel
- **Access control at the network layer**: NetworkPolicy enforces what the VPN pod can reach, independent of application-level auth
- **ReplicaSet-aware**: VPN clients can resolve all ReplicaSet member hostnames via cluster DNS, which is required for driver-level replica set negotiation

Alternatives considered:

| Approach | Rejected reason |
|---|---|
| NodePort on 27017 | Exposes DB port to internet |
| SSH tunnel | Requires shell access to a node; not scalable |
| Kubernetes API proxy | Not suited for persistent MongoDB connections |
| Service mesh egress gateway | Adds significant complexity; MongoDB protocol not HTTP |

---

## Why OpenVPN

- Mature, audited, widely supported across client platforms (Linux, macOS, Windows)
- TCP mode works through restrictive firewalls and proxies
- TLS-crypt (HMAC pre-auth) drops unauthenticated packets before TLS handshake
- Certificate-based auth with CRL support allows per-client revocation
- Established container image (`kylemanna/openvpn`) with minimal footprint

### Architecture Constraints

The primary constraint driving TCP mode is the load balancer tier. Many external load balancers — including Hetzner Load Balancer — support only TCP, not UDP. OpenVPN in UDP mode cannot be proxied through a TCP-only load balancer.

Beyond the infrastructure constraint, UDP is frequently blocked in restricted networks:

- Corporate firewalls that whitelist only TCP 443/80
- Proxy environments that intercept all non-TCP traffic
- Cloud load balancers without UDP listener support
- Aggressive NAT environments that drop stateless UDP flows

TCP mode ensures predictable behavior across all of these, at the cost of slightly higher per-packet overhead due to TCP-over-TCP encapsulation.

### Why not WireGuard / Tailscale

| Option | Reason rejected |
|---|---|
| WireGuard | UDP-only protocol, incompatible with TCP-only load balancers |
| Tailscale / ZeroTier | Requires external control plane, UDP transport, and may modify iptables routing rules on hosts |

Both WireGuard and overlay mesh solutions like Tailscale can dynamically modify the network configuration of cluster nodes (iptables rules, routing tables, kernel interfaces). This is undesirable in a Kubernetes cluster where the CNI plugin manages the network plane. Database access requires a predictable, fully controlled network path — a self-hosted OpenVPN deployment satisfies this without side effects on the node network stack.

---

## Kubernetes Implementation

### Resources

| Resource | File | Purpose |
|---|---|---|
| Namespace | `data-access/namespace-vpn.yaml` | Isolates VPN workload |
| Deployment | `data-access/openvpn-db-deployment.yaml` | Runs OpenVPN server pod |
| Service | `data-access/openvpn-db-service.yaml` | Exposes NodePort 31194 |
| Secret | `data-access/openvpn-db-secret.yaml` | PKI material and server config |
| NetworkPolicy | `data-access/networkpolicy-openvpn-db-egress.yaml` | Restricts egress to MongoDB only |

### Deployment

The OpenVPN pod requires `NET_ADMIN` capability and `privileged: true` to manage `tun` devices and configure `iptables` NAT rules. On startup it:

1. Enables IPv4 forwarding (`sysctl -w net.ipv4.ip_forward=1`)
2. Adds a MASQUERADE rule for VPN subnet `10.88.0.0/24`
3. Starts OpenVPN with the server config from the mounted Secret

The `/dev/net/tun` device is mounted from the host as a `CharDevice` volume.

### Service

NodePort service on TCP `31194` → pod port `1194`. An external TCP load balancer forwards TCP `:1194` to this NodePort. Protocol is TCP (not UDP) — see [Architecture Constraints](#architecture-constraints) below.

### Secret

The Secret (`openvpn-db-config`) contains the complete PKI and server configuration:

```
ca.crt          # CA certificate
server.crt      # Server certificate
server.key      # Server private key
dh.pem          # Diffie-Hellman parameters
ta.key          # TLS-crypt HMAC key
crl.pem         # Certificate Revocation List
server-tcp.conf # OpenVPN server configuration
```

In the repository, credential fields use placeholder values:

```
<CA_CERT>
<SERVER_CERT>
<SERVER_KEY>
<TA_KEY>
<DH_PARAMS>
<CRL>
```

PKI generation and management is handled by the Ansible automation referenced below.

### NetworkPolicy

`networkpolicy-openvpn-db-egress.yaml` applies an egress-only policy to the `openvpn-db` pod. Allowed egress:

- TCP `:27017` to MongoDB pods (`app.kubernetes.io/name: mongodb`) in namespaces: `dev`, `stage`, `prod-dubai`, `prod-france`, `prod-georgia`, `prod-greece`, `prod-spain`, `prod-ukraine`
- TCP `:27017` to pod CIDRs `10.233.0.0/18` and `10.233.64.0/18` (fallback for IP-based access)
- UDP/TCP `:53` to `kube-dns` in `kube-system` (required for DNS resolution)

All other egress from the VPN pod is denied.

### Server Configuration Highlights

```
proto tcp
server 10.88.0.0 255.255.255.0
tls-crypt /etc/openvpn/ta.key
auth SHA256
cipher AES-256-CBC
ncp-ciphers AES-256-GCM:AES-128-GCM:AES-256-CBC
tls-version-min 1.2
push "route 10.233.0.0 255.255.192.0"
push "route 10.233.64.0 255.255.192.0"
push "dhcp-option DNS 10.233.0.3"
push "dhcp-option DOMAIN cluster.local"
push "dhcp-option DOMAIN svc.cluster.local"
```

**Why two search domains are pushed:**

Both `cluster.local` and `svc.cluster.local` are pushed explicitly. macOS processes all pushed search domains correctly. Windows VPN clients sometimes only apply the first `DOMAIN` option, which means only `cluster.local` would be active. Pushing both ensures that Kubernetes service discovery — and therefore MongoDB replica set member resolution — works reliably regardless of the client operating system.

### OpenVPN Push Directives Considerations

The server pushes routes and DNS settings to all connected clients:

```
push "route 10.233.0.0 255.255.192.0"
push "route 10.233.64.0 255.255.192.0"
push "dhcp-option DNS 10.233.0.3"
push "dhcp-option DOMAIN cluster.local"
push "dhcp-option DOMAIN svc.cluster.local"
```

These directives enable cluster pod network access and internal service name resolution, but introduce operational considerations on the client side.

**Route injection**

Pushed CIDRs are written into the client's routing table for the duration of the VPN session. This can cause:

- Conflicts with existing corporate VPN routes using overlapping `10.x.x.x` ranges
- Routing precedence issues on developer machines with multiple active network interfaces

**DNS override**

Kubernetes DNS (`10.233.0.3`) is pushed as the resolver while the VPN is active. This may:

- Override the client's existing system resolver configuration
- Conflict with local or corporate DNS, causing resolution failures for non-cluster names
- Behave differently across operating systems (macOS, Windows, Linux handle pushed DNS with varying precedence)

**Mitigation**

Only the minimal routes required for MongoDB ReplicaSet access are pushed. No `redirect-gateway` directive is used, so client traffic is not fully tunneled through the VPN — only traffic destined for the Kubernetes pod CIDRs is routed through the tunnel.

---

## Operational Use Cases

Typical scenarios where this architecture is used:

- Database migrations requiring direct connection to a specific ReplicaSet primary or secondary
- ReplicaSet maintenance (stepDown, reconfiguration, member replacement)
- Operational debugging (oplog inspection, index analysis, slow query investigation)
- Disaster recovery procedures requiring direct data access
- Administrative database access outside of application connection pools

These operations require direct MongoDB connectivity that cannot be safely exposed through public endpoints.

---

## Monitoring Access Through VPN

This section demonstrates how the same VPN access pattern can be reused for monitoring systems.

The same OpenVPN gateway used for database access can also provide secure access to monitoring systems running outside the Kubernetes cluster.

A common pattern is Grafana deployed outside the cluster, connecting to Prometheus running inside it. Prometheus has no public endpoint — it remains internal to the cluster and is only reachable through the VPN tunnel.

```
Grafana Server
      |
OpenVPN Client
      |
VPN Tunnel
      |
Kubernetes Cluster
      |
Prometheus :9090
```

Grafana connects to Prometheus using the internal Kubernetes service name:

```
http://prometheus.monitoring.svc.cluster.local:9090
```

This requires the VPN client on the Grafana host to push the cluster pod CIDRs and DNS resolver — the same routes and DNS settings used for operational database access. No additional VPN infrastructure is needed; monitoring traffic shares the same tunnel.

Prometheus is never exposed publicly. Access to Prometheus is additionally restricted by Kubernetes NetworkPolicy rules applied to the VPN pod. Access is gated by VPN certificate authentication, with the same NetworkPolicy controls that apply to all VPN egress.

---

## When to use this architecture

This pattern is appropriate when:

- Databases run inside Kubernetes and must not expose public ports
- Administrative access is required from outside the cluster
- The infrastructure uses TCP-only load balancers
- The organization requires self-hosted networking without external control planes

This architecture is **not intended for application traffic** or general service connectivity.

It is designed specifically for **controlled operational access** to stateful infrastructure components such as databases.

---

## Client-side Considerations

Known client-side issues when connecting to MongoDB through this VPN.

### macOS DNS behavior

macOS treats `.local` as an mDNS (Bonjour) domain and may not route `*.cluster.local` queries to the VPN-pushed DNS server. This means Kubernetes service names can fail to resolve even when the VPN is connected.

Workaround — create per-domain resolver files:

```bash
sudo mkdir -p /etc/resolver

echo "nameserver 10.233.0.3" | sudo tee /etc/resolver/cluster.local
echo "nameserver 10.233.0.3" | sudo tee /etc/resolver/svc.cluster.local

sudo killall -HUP mDNSResponder
```

After this, Kubernetes DNS resolves correctly for both `cluster.local` and `svc.cluster.local` domains.

### Multiple VPN clients

Running multiple VPN clients simultaneously can cause DNS conflicts. A second VPN may intercept DNS queries before they reach the Kubernetes resolver, causing resolution failures for cluster-internal names. Disconnect other VPN clients before connecting to this gateway.

### MongoDB Compass behavior

MongoDB Compass may fail to connect when the connection string includes the `replicaSet` parameter. Compass attempts to resolve all ReplicaSet member hostnames during the initial handshake, which requires working cluster DNS for all members simultaneously — this fails on some client configurations.

`mongosh` and `TablePlus` handle this correctly and are the recommended clients for operational access.

If Compass must be used, connect to a single node directly:

```
mongodb://user:pass@mongodb-client.dev.svc.cluster.local:27017/db?authSource=admin
```

or force a direct connection:

```
mongodb://user:pass@mongodb-client.dev.svc.cluster.local:27017/db?authSource=admin&directConnection=true
```

---

## Security Considerations

**Principle of least privilege (network layer)**
NetworkPolicy ensures the VPN pod has no egress beyond MongoDB and DNS. Compromising the VPN server does not grant lateral movement to arbitrary cluster resources.

**Pre-auth packet filtering**
`tls-crypt` with a shared HMAC key authenticates packets before TLS negotiation. Unauthenticated traffic is dropped without processing, reducing exposure to TLS exploits.

**Certificate revocation**
CRL is mounted from the Secret. Revoking a client certificate requires updating `crl.pem` in the Secret and restarting the pod. For higher operational velocity, consider automating CRL rotation via the Ansible playbooks.

**Privileged pod**
The deployment requires `privileged: true` for `tun` device management. This is a known trade-off. Mitigation: the pod runs as `nobody:nogroup` after initialization, and the namespace isolates the workload.

**Secret management**
PKI material is stored in a Kubernetes Secret. For production, consider integrating with a secrets manager (Vault, Sealed Secrets, or External Secrets Operator) to avoid storing key material in cluster etcd unencrypted.

**Single replica**
The Deployment runs 1 replica. OpenVPN state (`ipp.txt`, status log) is written to `/tmp` (ephemeral). Reconnection after pod restart is handled by VPN clients automatically, but there is a brief downtime window during restarts.

---

## Engineering Trade-offs

This architecture intentionally prioritizes security and operational control over simplicity.

Running OpenVPN inside Kubernetes introduces additional operational responsibility, including PKI management, certificate rotation, and VPN server maintenance.

However, it eliminates the need to expose database services to the internet and provides a predictable network access model for administrative operations.

For environments where database security is critical, this trade-off is often justified.

---

## Repository Structure

The repository is organized into two logical components:

- **data-access** — primary architecture for secure MongoDB administrative access
- **monitoring** — optional extension demonstrating VPN-based access to monitoring systems (Prometheus)

```
.
├── data-access/
│   ├── namespace-vpn.yaml
│   ├── openvpn-db-deployment.yaml
│   ├── openvpn-db-service.yaml
│   ├── openvpn-db-secret.yaml
│   └── networkpolicy-openvpn-db-egress.yaml
│
├── monitoring/
│   ├── openvpn-monitoring-deployment.yaml
│   ├── openvpn-monitoring-service.yaml
│   ├── openvpn-monitoring-secret.yaml
│   ├── networkpolicy-openvpn-monitoring-egress.yaml
│   └── openvpn-client-monitoring.conf
│
└── README.md
```

Apply order:

```bash
kubectl apply -f data-access/namespace-vpn.yaml
kubectl apply -f data-access/openvpn-db-secret.yaml
kubectl apply -f data-access/openvpn-db-deployment.yaml
kubectl apply -f data-access/openvpn-db-service.yaml
kubectl apply -f data-access/networkpolicy-openvpn-db-egress.yaml
```

---

## External Dependencies

**OpenVPN PKI automation**

Certificate authority setup, server/client certificate issuance, and CRL management are handled by:

[ansible-openvpn-server](https://github.com/SergeyZaika/ansible-openvpn-server)

That repository covers OpenVPN installation, EasyRSA PKI initialization, and client profile generation. The PKI output (CA cert, server cert/key, DH params, TA key, CRL) is what populates the Kubernetes Secret in this repository.
