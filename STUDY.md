# Study: Exclave Project & Routing Rules with TrustTunnel

## Overview

**Exclave** (<https://github.com/dyhkwong/Exclave>) is an Android proxy client forked
from the archived SagerNet project. It uses a custom, heavily-modified fork of
**V2Ray core** (`dyhkwong/v2ray-core`) as its networking engine.

- **Language**: Kotlin 78%, Java 11%, Go 10%
- **License**: GPL-3.0
- **Stars**: ~1.8k
- **TrustTunnel support**: Added in v0.17.11-beta.0 (January 28, 2026)

Exclave proves that **full routing rules work with TrustTunnel**, completely
independently of the VPN/proxy protocol being used.

---

## Architecture

Exclave is a two-layer system:

```
┌──────────────────────────────────────────────────────┐
│            Android/Kotlin Layer (app/)                │
│                                                      │
│  Protocol Beans       Routing Rules     Config Builder│
│  ┌──────────────┐    ┌─────────────┐   ┌───────────┐│
│  │TrustTunnelBn │    │ RuleEntity  │   │ConfigBuildr││
│  │VMessBean     │    │  domains    │   │  beans →   ││
│  │TrojanBean    │    │  ip         │──▶│  V2Ray JSON││
│  │ShadowsocksBn │    │  port       │   │  outbounds ││
│  │WireGuardBean │    │  packages   │   │  + routing ││
│  │...           │    │  ssid       │   └───────────┘│
│  └──────────────┘    │  → outbound │                 │
│                      └─────────────┘                 │
└──────────────────────────────────────────────────────┘
                       │
                       ▼ V2Ray JSONv4 config
┌──────────────────────────────────────────────────────┐
│          V2Ray Core (Go, dyhkwong's fork)             │
│                                                      │
│  Inbound → Router (rule matching) → Outbound Handler │
│            ├─ domain match?                          │
│            ├─ IP match?          tag:"proxy" → HTTP  │
│            ├─ port match?                   CONNECT  │
│            ├─ app/UID match?     tag:"direct"→Freedom│
│            └─ SSID match?        tag:"block" →Blackhl│
└──────────────────────────────────────────────────────┘
```

---

## TrustTunnel Implementation Details

### Protocol Nature

As Exclave's developer correctly identifies:

> "TrustTunnel is in fact a proxy protocol, rather than a so-called 'VPN protocol'.
> For TCP, it is a standard HTTP/2 CONNECT tunnel or HTTP/3 CONNECT tunnel.
> For UDP, it uses a private UDP over TCP magic address protocol."

### TrustTunnel Protocol Spec (from PROTOCOL.md)

| Traffic | Method | Details |
|---------|--------|---------|
| **TCP** | HTTP CONNECT | `CONNECT host:port HTTP/2` per-stream bidirectional tunnel |
| **UDP** | Magic address `_udp2` | Single multiplexed stream, binary framing: 4B length + 16B srcAddr + 2B srcPort + 16B dstAddr + 2B dstPort + payload |
| **ICMP** | Magic address `_icmp` | Single multiplexed stream, ping request/reply framing |
| **Health** | Magic address `_check` | `CONNECT _check`, 200 = healthy |

All multi-byte integers use network byte order (big-endian). IPv4 addresses are
zero-padded to 16 bytes.

### Data Model (TrustTunnelBean.java)

```java
public class TrustTunnelBean extends AbstractBean {
    public String protocol;         // "https" (HTTP/2) or "quic" (HTTP/3)
    public String username;         // authentication
    public String password;
    public String sni;              // TLS Server Name Indication
    public String certificate;      // custom CA cert (PEM)
    public String utlsFingerprint;  // browser TLS fingerprint emulation
    public Boolean allowInsecure;   // skip TLS verification
    // inherited: serverAddress, serverPort
}
```

### Config Builder Mapping

The `ConfigBuilder.kt` maps TrustTunnel to **existing V2Ray outbound types**:

- `protocol: "https"` → V2Ray **`http` outbound** + TLS stream settings (HTTP/2 CONNECT)
- `protocol: "quic"` → V2Ray **`http3` outbound** (HTTP/3 CONNECT)

**No new protocol engine was needed.** TrustTunnel maps directly to V2Ray's
existing HTTP CONNECT proxy client implementation.

### Deep Link Format

TrustTunnel configs can be shared via `tt://` URIs using TLV (Tag-Length-Value)
binary encoding with Base64 URL-safe encoding. Supported tags include:
Version, Hostname, Addresses, CustomSNI, HasIPv6, Username, Password,
SkipVerification, Certificate, UpstreamProtocol, AntiDPI, ClientRandomPrefix.

---

## Routing Rules System

### Rule Types (from RuleEntity.kt)

| Condition | Description |
|-----------|-------------|
| `domains` | Domain matching: exact, suffix, keyword, regex, `geosite:` tags |
| `ip` | Destination IP/CIDR matching, `geoip:` tags |
| `port` | Destination port or port range |
| `sourcePort` | Source port matching |
| `network` | `"tcp"`, `"udp"`, or both |
| `source` | Source IP address matching |
| `protocol` | Sniffed protocol (http, tls, bittorrent) |
| `packages` | Android app package names (per-app routing) |
| `ssid` | Wi-Fi network name |
| `networkType` | wifi, data, ethernet, bluetooth, usb, satellite |
| `reverse` | Invert the match |

### Rule Evaluation

- Rules are evaluated **top-to-bottom** (first match wins)
- Within a single rule, all conditions are **AND**-ed
- Each rule routes to an **outbound tag**: proxy, direct, or block
- `domainStrategy` controls DNS resolution behavior for unmatched domains

### Geo Data

- `geoip.dat` — GeoIP database (MaxMind-derived, binary protobuf)
- `geosite.dat` — GeoSite database (from v2fly/domain-list-community, binary protobuf)
- Custom route assets can be added and managed

### Example V2Ray JSON Config (generated by ConfigBuilder)

```json
{
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "domain": ["geosite:category-ads-all"],
        "outboundTag": "block"
      },
      {
        "type": "field",
        "domain": ["geosite:cn"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": ["geoip:cn", "geoip:private"],
        "outboundTag": "direct"
      }
    ]
  },
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "http",
      "settings": {
        "servers": [{
          "address": "vpn.example.com",
          "port": 443,
          "users": [{"user": "username", "pass": "password"}]
        }]
      },
      "streamSettings": {
        "security": "tls",
        "tlsSettings": { "serverName": "vpn.example.com" }
      }
    },
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ]
}
```

---

## Key Architectural Insight: Protocol-Agnostic Routing

The routing engine and the outbound protocol are **completely separate layers**.

V2Ray's outbound abstraction:

```go
// Every proxy protocol implements this same interface
type Outbound interface {
    Process(ctx context.Context, link *transport.Link, dialer internet.Dialer) error
}
```

The router works at the **connection metadata level** (domain, IP, port, protocol,
network type, UID) and produces a **tag**. The outbound tagged with that tag handles
the actual tunneling using whatever protocol it implements.

**The same routing rules work identically whether the outbound is VLESS, WireGuard,
Shadowsocks, or TrustTunnel.** The routing layer is completely protocol-agnostic.

For TrustTunnel:
1. `TrustTunnelBean` creates a V2Ray outbound with `protocol = "http"/"http3"` and a unique tag
2. Routing rules reference that tag via `outboundTag`
3. Router dispatches matching traffic to the tagged outbound
4. The HTTP/HTTP3 CONNECT client establishes the TrustTunnel connection

**No TrustTunnel-specific routing code was needed.** Exclave reuses its existing
HTTP and HTTP3 CONNECT outbound implementations.

---

## Implications for Trusty

### Current Limitation

Trusty wraps the TrustTunnel CLI, which only supports a flat `exclusions` array.
The **CLI is the bottleneck**, not the protocol.

### Possible Approaches

#### Option A: Embed a routing engine (recommended)

Embed sing-box or V2Ray core as a Go library in Trusty's backend:

1. Run a local TUN/SOCKS proxy with a full routing engine
2. Configure TrustTunnel server as an HTTP/2 CONNECT outbound
3. Get full routing rules for free (domains, IPs, geosite, geoip, everything)
4. No dependency on TrustTunnel CLI at all

**Pros**: Full feature parity with Exclave, proven architecture
**Cons**: Significant engineering effort, larger binary, v2ray/sing-box dependency

#### Option B: Local proxy layer in front of TrustTunnel CLI

1. Trusty runs a lightweight local routing proxy
2. Apply domain/IP matching rules locally
3. "Direct" traffic bypasses TrustTunnel entirely
4. "Proxy" traffic goes through the TrustTunnel CLI

**Pros**: Less invasive, keeps TrustTunnel CLI
**Cons**: Limited rule types, complex process management

#### Option C: Implement TrustTunnel protocol natively in Go

Since TrustTunnel is just HTTP/2 CONNECT + simple UDP framing:

1. Implement a Go TrustTunnel client (HTTP/2 CONNECT + UDP magic address)
2. Integrate with a routing engine
3. Full control over both routing and tunneling

**Pros**: Maximum control, smaller dependency tree
**Cons**: Must implement and maintain the protocol, UDP framing complexity

### Verdict

Exclave proves conclusively that **the TrustTunnel CLI's `exclusions` limitation
is a CLI limitation, not a protocol limitation**. The protocol itself is just a
standard HTTP proxy — any routing engine can use it as an outbound.

---

## Related Projects

| Project | Description |
|---------|-------------|
| [Exclave](https://github.com/dyhkwong/Exclave) | Android proxy client (SagerNet fork) with TrustTunnel support |
| [dyhkwong/v2ray-core](https://github.com/dyhkwong/v2ray-core) | Custom v2ray-core fork used by Exclave |
| [Throne](https://github.com/throneproj/Throne) | Desktop proxy client wrapping sing-box (C++/Qt + Go) |
| [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel) | Official protocol spec and server implementation |
| [TrustTunnelClient](https://github.com/TrustTunnel/TrustTunnelClient) | Official CLI client |
| [v2fly/domain-list-community](https://github.com/v2fly/domain-list-community) | Community-maintained domain categorization data |
| [SagerNet/sing-geosite](https://github.com/SagerNet/sing-geosite) | Compiled sing-box format geosite databases |
| [SagerNet/sing-geoip](https://github.com/SagerNet/sing-geoip) | Compiled sing-box format geoip databases |

---

## Sources

- <https://github.com/dyhkwong/Exclave>
- <https://github.com/dyhkwong/Exclave/wiki/Configuration>
- <https://github.com/dyhkwong/Exclave/wiki/Route>
- <https://github.com/dyhkwong/Exclave/releases>
- <https://github.com/TrustTunnel/TrustTunnel/blob/master/PROTOCOL.md>
- <https://github.com/dyhkwong/v2ray-core>
- <https://f-droid.org/packages/com.github.dyhkwong.sagernet/>
