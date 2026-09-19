# Andromeda Protocol Specification

Version: v0.1 draft

## 1. Purpose

Andromeda is a tunneling protocol for carrying IPv4 traffic over a TCP connection that presents itself as ordinary TLS 1.3 / HTTPS traffic. The protocol is intended to resist passive and active DPI analysis while preserving the external appearance of a normal encrypted web session.

The central principle is simple: do not impersonate HTTPS. Use real TLS 1.3, with correctly shaped traffic, a valid certificate chain, and protocol behavior consistent with a normal HTTPS session. The tunnel logic is carried within the TLS application data stream.

## 2. Threat model

### Adversary

- ISP with passive DPI analysis
- ISP with active probing and manipulation
- National firewall performing heuristics and ML classification
- Enterprise firewall with SNI and TLS fingerprint blocking

### Adversary capabilities

- Passive analysis of packet sizes, timing, and traffic structure
- Active probing into the server endpoint
- Blocking by IP, port, SNI, and TLS fingerprint
- Replay attacks against 0-RTT features
- MITM at the network layer

### Out of scope

- Attacks on cryptographic primitives
- Compromise of end devices
- Global correlation across the two tunnel endpoints

## 3. Architecture

The protocol stack is:

```text
[IP packet] -> [Andromeda frame] -> [TLS 1.3 record] -> [TCP] -> [IP]
```

Layers:

1. Transport: TCP
2. Security and disguise: real TLS 1.3
3. Tunnel layer: Andromeda frames
4. Payload: IP packets

The key design decision is that the tunnel is not a fake TLS session. It is a genuine TLS 1.3 connection using real records and normal HTTPS behavior. This preserves correct packet sizes, timing patterns, and certificate semantics required to blend into standard web traffic.

## 4. Frame format

The frame is transmitted inside a TLS record, as application data. The header is intentionally variable and extensible.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Ver  |  Type |     Flags     |         Stream ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Length               |         Reserved              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         TLV Options ...                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Payload ...                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Field definitions

- Ver (4 bits): protocol version. v0.1 = 1.
- Type (4 bits): DATA, PING, PONG, CLOSE, OPEN_STREAM, CLOSE_STREAM, ERROR
- Flags (8 bits): ACK_REQ, FRAG_MORE, FRAG_LAST, PADDING, URGENT
- Stream ID (16 bits): stream identifier
- Sequence Number (32 bits): sequencing value for fragmentation and reassembly
- Length (16 bits): payload length
- Reserved (16 bits): reserved for future use
- TLV options: optional metadata such as padding, timestamp, ACK, and MTU probe information
- Payload: the encapsulated IP packet or tunnel metadata

### Encryption

Encryption is provided exclusively by TLS 1.3. An additional AES-GCM layer is intentionally not introduced. Double encryption adds unnecessary overhead and creates visible size artifacts that are incompatible with the anti-DPI design goal.

## 5. Connection states

```text
CLOSED -> TCP_CONNECTING -> TLS_HANDSHAKE -> ANDROMEDA_INIT
      -> ESTABLISHED <-> (DATA, STREAMS, PING)
      -> CLOSING -> CLOSED
```

### Handshake sequence

1. TCP connection is established.
2. TLS 1.3 handshake occurs in full 1-RTT mode on first connection; 0-RTT is allowed only for valid resumptions.
3. HELLO frames are exchanged over the TLS channel to negotiate version, extensions, MTU, and client authentication.
4. The session transitions to ESTABLISHED.

### Closing sequence

- Explicit CLOSE frame is sent
- TLS close_notify is performed
- TCP FIN is sent afterward

### Session migration

Session migration is not supported in v0.1. The tunnel remains bound to the existing TCP 4-tuple lifecycle.

## 6. Cryptography

- Transport encryption: TLS 1.3
- AEAD: AES-GCM (128 or 256)
- Key exchange: X25519
- Forward secrecy: yes, inherent to TLS 1.3
- Authentication: self-signed certificate with client pinning
- Stub mode: the server maintains a separate genuine Let's Encrypt certificate for a decoy website
- 0-RTT: allowed for session resumption with PSK and mandatory replay protection

### 0-RTT replay policy

- Server keeps a window of previously used 0-RTT tokens
- Token lifetime is capped at 5 minutes
- Reuse triggers immediate connection drop without response

## 7. Anti-DPI and camouflage

The tunnel is presented as a normal HTTPS endpoint on port 443.

### Authentication failure behavior

If authentication fails, the server does not send an RST. Instead, it forwards the TCP connection to a real decoy site running nginx with a valid Let's Encrypt certificate. The peer obtains a benign HTML page. To DPI, the result is indistinguishable from an ordinary HTTPS website.

RST is explicitly forbidden because it is an atypical and suspicious behavior for a legitimate HTTPS endpoint.

### Active probing resistance

- Decoy site is served using a real certificate
- Timing is shaped to match legitimate HTTPS behavior
- TLS response behavior is valid and non-suspicious
- A probe cannot reliably distinguish the stub from genuine traffic

### Packet-size and timing shaping

- Padding is inserted via TLV options
- Payload is aligned to multiples of 128 bytes
- Jitter is applied to send timing
- The tunnel avoids patterns that look like request-response tunneling behavior

## 8. Multiplexing and addressing

- Multiplexing is based on Stream ID
- Each stream has flow-level state and a receive window
- IPv4 is supported in v0.1
- IPv6 and domain addresses are not included in v0.1 and are planned for v0.2
- OPEN_STREAM and CLOSE_STREAM are used to establish and terminate flows

## 9. MTU and fragmentation

Andromeda uses in-protocol fragmentation.

- FRAG_MORE and FRAG_LAST flags indicate continuation states
- Sequence numbers are used for reassembly
- Reassembly key is (Stream ID, Sequence Number)
- Fragment timeout is 5 seconds
- Expired fragments are dropped

## 10. Extensibility and versioning

- Version is encoded in the Ver field
- TLV options allow future features and metadata
- HELLO frames negotiate protocol capabilities
- Unknown TLV types are ignored to preserve forward compatibility

## 11. Scope of v0.1

Included in v0.1:

- TLS 1.3 over TCP
- Andromeda framing
- Basic single-stream tunnel model
- IPv4 payload support
- Fragmentation and reassembly
- Authentication failure stub
- 0-RTT with replay protection
- Padding and jitter shaping

Not included in v0.1:

- IPv6 and domain addressing
- Encrypted DNS
- Session migration
- HTTP/3 or QUIC
- Full many-stream multiplexing
- Custom congestion control, which remains delegated to the host TCP stack

## 12. Development stages

### v0.1 MVP

- SPEC.md describing protocol format, state machine, and cryptography
- TLS 1.3 handshake with pinning
- Framing and serialization/parsing
- Single data stream
- Fragmentation and reassembly
- Stub fallback for failed authentication
- Padding behavior

### v0.2

- Full stream multiplexing
- IPv6 and domain support
- Complete 0-RTT

### v0.3

- Timing-correlation resistance
- DPLPMTUD
- Performance optimization

### v1.0

- Cryptographic audit
- Formal state verification
- Public specification release

## 13. Readiness criteria for v0.1

The v0.1 release is considered ready when:

- The tunnel can transport IP packets end-to-end
- Traffic is indistinguishable from HTTPS under tcpdump and DPI heuristics
- Active probes receive a valid HTML stub page
- 0-RTT replay is rejected successfully
- Fragmentation works at MTU 1280
- Padding aligns sizes to expected distributions
- SPEC.md describes format, state transitions, and cryptographic properties

## 14. Test plan

### Unit tests

- Frame parse/serialize correctness
- TLV option processing
- Fragmentation and reassembly logic

### Integration tests

- Handshake and tunnel establishment
- ICMP/TCP/UDP transport through the tunnel
- Graceful closure

### Security tests

- 0-RTT replay rejection
- MITM without pinning
- Active probing against the stub

### DPI simulation tests

- Compare the outbound fingerprint against a real HTTPS flow
- Ensure no characteristic tunnel signatures are visible

## 15. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Self-signed certificate visible to DPI | Use an LE-backed decoy site and client pinning |
| 0-RTT replay | Token windows and short TTL |
| Timing correlation | Jitter and padding |
| Visible size artifacts in TLS records | Align records to 128-byte chunks |
| Active probing | Valid stub page and consistent timing |
| TCP head-of-line blocking | Considered an accepted v0.1 limitation |

## 16. License and publication

- License: MIT or Apache 2.0
- SPEC.md must remain in the repository root
- README must describe architecture, deployment, and run examples
- CI must build the project and run unit tests
- A demo should compare packet capture before and after the tunnel, using live HTTPS traffic as the baseline

## 17. Security notes

This protocol is intentionally designed for concealed transport. However, no protocol is invulnerable to endpoint compromise or traffic-correlation beyond the modeled threat surface. The implementation should treat all anti-DPI properties as best-effort network camouflage, not as a guarantee against all possible analysis.

## 18. Summary

Andromeda is a TLS 1.3 based tunnel designed to carry IPv4 packets over standard HTTPS-like traffic. It preserves the external shape of real web traffic while carrying a custom framing layer inside TLS payload data. The v0.1 scope focuses on a single-stream IPv4 tunnel with fragmentation, padding, stub fallback behavior, and replay-resistant 0-RTT semantics.

This document is intended as the canonical specification for the initial implementation and future extension points.
