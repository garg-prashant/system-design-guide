# Protocols: TCP/UDP, HTTP/2, gRPC, WebSockets

## TCP vs UDP

| | TCP | UDP |
|---|-----|-----|
| **Reliability** | Reliable; retransmits; ordered. | Best-effort; no guarantee. |
| **Connection** | Connection-oriented; handshake. | Connectionless. |
| **Use case** | HTTP, gRPC, DB connections, file transfer. | Video/audio streaming, gaming, DNS (short queries). |
| **Overhead** | Higher (headers, acks, flow control). | Lower. |

**Interview:** Use TCP when you need reliable, ordered delivery; UDP when latency and throughput matter more than every packet (e.g. real-time media).

---

## HTTP/1.1 vs HTTP/2

| Aspect | HTTP/1.1 | HTTP/2 |
|--------|----------|--------|
| **Connections** | Multiple connections per host (often 6). | Single connection; **multiplexing** of streams. |
| **Headers** | Plain text; repeated per request. | Binary; **HPACK** compression; header table. |
| **Server push** | No. | Server can push resources. |
| **Head-of-line** | One slow request can block others on same connection. | Streams multiplexed; one slow stream doesn’t block others. |

**Interview:** “We use HTTP/2 so many requests share one connection and we avoid connection limit and head-of-line blocking at the HTTP layer.”

---

## gRPC

- **Built on:** HTTP/2; Protocol Buffers (binary).
- **Model:** Client calls a method on a server; strongly typed contracts (.proto).
- **Streaming:** Unary, server stream, client stream, bidirectional.
- **Use case:** Service-to-service; microservices; when you want strict contracts and efficient binary encoding.

**Trade-offs:** Not browser-native (need grpc-web or proxy); tooling and debugging differ from REST. Great for backend-to-backend.

---

## WebSockets

- **Purpose:** Full-duplex, long-lived connection over a single TCP connection. Initiated via HTTP upgrade.
- **Use case:** Real-time: chat, live feeds, gaming, dashboards.
- **Interview:** “We use WebSockets for real-time updates so the client doesn’t poll; we run them behind a load balancer that supports WebSocket (sticky or connection upgrade).”

---

## DNS and TLS (Brief)

- **DNS:** Resolves name → IP; TTL (e.g. 300s); critical for failover and CDN. Cache at OS and application.
- **TLS:** Encrypts and authenticates; 1–2 RTT for handshake. Terminate at LB or gateway to centralize certs and reduce backend load.
