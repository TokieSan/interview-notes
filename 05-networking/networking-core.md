# Networking Core (C2)

Use the layers as a diagnostic ladder, not a recitation. Each item: mechanism, then what it breaks in production.

*Feeds: G1 heavily, G2, A4.*

**C2.1 — Layers, used properly.**
Don't recite OSI. Use it as a *diagnostic ladder*: link (is the NIC up, are there errors/drops?) → network (is there a route, does ICMP get there?) → transport (does the port accept, does the handshake complete?) → TLS (does the cert validate, does SNI match?) → application (does the request return, and what status?). Say it that way and the "explain the OSI model" question becomes a demonstration of method rather than recall.

**C2.2 — TCP handshake and teardown.**
SYN → SYN-ACK → ACK. Server-side there are two queues: the SYN queue (half-open, `tcp_max_syn_backlog`) and the accept queue (established, awaiting `accept()`, sized by the `listen()` backlog and `somaxconn`). Teardown: FIN → ACK → FIN → ACK, the closing side entering `TIME_WAIT` for 2×MSL.
*Consequence, and this is the payload:* accept-queue overflow silently drops or resets connections while the server looks healthy — `ss -lnt` shows `Recv-Q` above `Send-Q`, and `nstat | grep -i listendrop` confirms. That is the difference between "the app is slow" and "the app never got the connection."

**C2.3 — TIME_WAIT.**
Held by the side that closes first, to absorb delayed duplicates and to guarantee the final ACK can be retransmitted. Thousands of `TIME_WAIT` sockets are normal on a busy client; they only hurt when you exhaust the ephemeral port range to a *single* destination tuple (`net.ipv4.ip_local_port_range`).
*Consequence:* the fix is connection pooling / keepalive, not `tcp_tw_recycle` (removed — it broke NAT). `tcp_tw_reuse` is safe only for outbound connections.

**C2.4 — Congestion control and flow control are different things.**
Flow control = receiver window (`rwnd`), protects the *receiver's* buffer. Congestion control = `cwnd`, protects the *network*. Effective window = min of the two. Classic loss-based (CUBIC) treats loss as congestion; BBR models bottleneck bandwidth and RTT instead.
*Consequence:* on a long fat pipe, throughput is bounded by `window / RTT`. A 64 KB window over a 100 ms RTT caps you at ~5 Mbps regardless of a 10 Gbps link. Window scaling exists for exactly this. Useful in G3 capacity work.

**C2.5 — MTU, MSS, and PMTU black holes.**
MTU 1500 typical, MSS = MTU − 40 (IPv4+TCP headers). Path MTU discovery sets DF and relies on ICMP "fragmentation needed" coming back. Firewalls that drop all ICMP break this.
*Consequence:* the signature is *small requests succeed, large ones hang.* Handshake works, then the connection stalls on the first full-size segment. Genuinely one of the best things you can name in a debugging round because almost nobody does.

**C2.6 — TCP vs UDP, past the obvious.**
TCP: ordered, reliable, congestion-controlled, connection-oriented, head-of-line blocking within a stream. UDP: datagram, unordered, no delivery guarantee, no congestion control — you build what you need on top. QUIC is UDP-based and solves TCP's HOL blocking across multiplexed streams while keeping congestion control and adding built-in TLS 1.3.
*Consequence:* HTTP/2 multiplexes over one TCP connection, so one lost packet stalls *all* streams. HTTP/3 (QUIC) fixes that. If asked "why HTTP/3", that's the answer, not "it's faster."

**C2.7 — DNS resolution path.**
`getaddrinfo()` → NSS (`/etc/nsswitch.conf`) → possibly `/etc/hosts`, then the stub resolver (`/etc/resolv.conf`) → recursive resolver → root → TLD → authoritative. Caching at every hop, bounded by TTL; negative caching via SOA minimum.
*Consequence:* DNS is the classic "it's always DNS" because failures are *partial and delayed* — stale caches, split-horizon views, `search` domain expansion, `ndots` in Kubernetes causing 4× lookups per query. Ask "does it fail by name or by IP?" as the first branch in any connectivity investigation.

**C2.8 — TLS handshake.**
1.2: ClientHello → ServerHello + cert → key exchange → Finished (2 RTT). 1.3: one RTT, key share sent in ClientHello, plus 0-RTT resumption (with replay caveats). SNI carries the hostname in the clear so a server can pick a cert; ALPN negotiates the protocol.
*Consequence:* certificate expiry is a scheduled outage that everyone schedules and nobody notices. Chain issues manifest as "works in browser, fails in curl" (browsers fetch missing intermediates, most clients don't). Clock skew breaks validity windows.

**C2.9 — The tool ladder, and what each proves.**
`ping` — L3 reachability and RTT (but ICMP is often deprioritised or blocked; failure ≠ down). `traceroute`/`mtr` — path and per-hop loss (mid-path loss with no end loss is usually rate-limited ICMP, not real). `dig +trace` — resolution path, separates cache from authority. `ss -tnp` — socket states, queues, owning process. `tcpdump -i any -nn 'port 443'` — ground truth, is the packet on the wire. `curl -v --resolve` — bypass DNS to isolate it. `ip route get <ip>` — which route/interface is actually chosen.
*Framing:* name the tool *and the hypothesis it kills.* Google's rubric rewards knowing what evidence you're gathering rather than reeling off commands.

**C2.10 — Detecting a transparent proxy.**
TTL anomalies, unexpected `Via`/`X-Forwarded-For` headers, TLS cert issuer differing from expectation (MITM re-signing), the connection succeeding to an IP that shouldn't be listening, or a `tcpdump` on the destination showing no arriving packets while the client sees an established connection. That last one is decisive.

**C2.11 — Load balancer types.**
L4 (connection-level, NAT or DSR, cheap, protocol-agnostic, no visibility into requests) vs L7 (terminates and re-issues, can route by path/header, retry, and do health checks that mean something, but costs CPU and adds a hop). Algorithms: round-robin, least-connections, EWMA-latency, consistent hashing (for cache affinity), power-of-two-choices (near-optimal with minimal state — good thing to name).
*Consequence:* L4 health checks tell you a port is open. L7 health checks tell you the service works. Most "half the fleet was serving 500s and the LB kept sending traffic" incidents are an L4 check on an L7 problem.

**C2.12 — Anycast vs GeoDNS.**
Anycast: same IP announced from many locations, BGP picks the closest — fast failover, no client state, but a routing change can break long-lived TCP connections mid-flight. GeoDNS: resolver location determines the answer — respects TTLs poorly, sees the *resolver's* location not the user's.

## HTTP status codes worth knowing cold

2xx success, 3xx redirect, 4xx client error (400/401/403/404/429), 5xx server error. **502 vs 503 vs 504:** 502 bad gateway (upstream returned garbage), 503 service unavailable (overloaded/down), 504 gateway timeout (upstream too slow).

## Packet loss — where and why packets die

Be able to enumerate _where and why_ packets die on different topologies:

- **Congestion / buffer overflow:** router/switch queue fills → tail drop. Most common cause on the internet. Bufferbloat = opposite failure (excess buffering → latency, not loss).
- **Physical layer / bit errors:** noisy links, faulty cables, attenuation, EMI. Wireless especially — interference, weak signal, collisions.
- **Wireless-specific:** half-duplex collisions, hidden-node problem, signal fading, contention (CSMA/CA backoff).
- **Wired LAN:** duplex mismatch, bad NIC/cable, switch buffer exhaustion under microbursts.
- **Faulty/misconfigured hardware:** overloaded router CPU, bad line cards.
- **MTU mismatch / fragmentation:** oversized packets dropped when DF bit set (PMTUD blackholes when ICMP is filtered).
- **Policing / rate limiting / QoS:** traffic exceeding a policer is dropped by design.
- **Routing issues:** black holes, routing loops (TTL expiry), asymmetric routes, flapping links, BGP misconfig.
- **Overloaded end host:** receive buffer full, kernel drops (`netstat -s`, `ifconfig` drops/overruns).
- **Firewall / ACL drops** (silent).
- **TCP vs UDP:** TCP retransmits + congestion-controls (loss → backoff via AIMD); UDP just loses it (app must handle). Trading systems often use UDP multicast → loss handling matters (gap-fill, sequence numbers, arbitration between A/B feeds).
- **Diagnosis tools:** `ping` (loss %), `mtr`/`traceroute` (where), `tcpdump`/Wireshark (what), interface counters, `ss -i`.

## Networking for low-latency / trading systems

Reports show networking goes deep for some Systems/onsite rounds (TCP internals, multicast, "design packet routing between a hub and node servers," "networking protocols and reliability").

- **TCP internals:** 3-way handshake (SYN/SYN-ACK/ACK), reliability via sequence numbers + ACKs + retransmission, **flow control** (receive window) vs **congestion control** (slow start, congestion avoidance, fast retransmit/recovery; algorithms like Cubic/BBR), Nagle's algorithm + delayed ACK (and why `TCP_NODELAY` matters for latency-sensitive trading traffic), `TIME_WAIT`. TCP vs UDP: ordered/reliable/stream vs unordered/unreliable/datagram-and-lower-latency.
- **Multicast:** one-to-many via group addresses + IGMP; **exchanges fan out market data over UDP multicast** because it scales to many subscribers without per-receiver state — directly relevant to HRT's world. Reliability is layered on top (sequence numbers + gap-fill/retransmit request), since UDP doesn't guarantee delivery.
- **Routing/reliability design Q:** structure it — addressing/topology, transport choice (TCP for reliability vs UDP/multicast for fan-out latency), failure handling (heartbeats, timeouts, retries, idempotency), ordering/dedup (sequence numbers), backpressure, and redundancy (dual feeds / A-B arbitration, which is how real market-data plants work).
- Inspect: `ss`, `tcpdump`/`wireshark`, `ip route`, `netstat`, `ethtool`.

## One-line summary

**Net:** two accept queues, overflow is silent · TIME_WAIT is on the closer · window/RTT bounds throughput · small OK + large hang = PMTU · fails by name not IP = DNS · L4 checks prove a port, L7 checks prove a service.
