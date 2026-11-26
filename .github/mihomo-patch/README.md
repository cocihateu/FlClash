# Mihomo fallback policy

ClashMetaForAndroid (CMA) owns the canonical Go implementation under
`core/src/foss/golang/clash`. FlClash carries the same fallback group and provider
health-check integration under `core/Clash.Meta`.
Maintain both ports together; preserve FlClash's upstream-specific `UsePosition`
constructor argument and adapt each patch's hunk offsets to its pinned source.
SimpleXray implements the equivalent policy through its existing
Kotlin `FallbackRelayServer`, `GroupRuntimeController`, and `FallbackPolicy`.

The [build instructions](../../.agents/commands.md#building) apply both patches in full to the pinned core:
first `enhance-fallback.patch` for the group policy and the provider's health-check
completion callback, then `fallback-udp.patch` for destination-specific UDP routing.
The manual `build` workflow compiles the group package with `go build ./adapter/outboundgroup/...`.

## Shared contract

- Each TCP write has its own RTT-based 4–10 second window to complete. Completed
  writes can extend the response deadline to at most two windows from the first
  unanswered write. If that deadline expires while an upload is still making
  progress, the missing reply is weak evidence: the next connection to that host
  can use a backup while the current upload continues.
- When a TCP stream receives a push after its idle window, the first write within one window
  could be a reply to the push or a new request. Silence after that write gives
  one weak signal for that destination, then the stream returns to idle. Each
  later push after an idle window gets the same treatment, including repeated heartbeats.
  A blocked write reaching its own deadline, a hard I/O failure, a genuine
  unanswered request on a quiet stream, or independent confirmation that the
  serving member failed still follows the usual retirement rules.
- A TCP response stall, ordinary EOF with a request outstanding (even after an
  earlier reply), a strong I/O error, or UDP silence can change the next route
  **for the affected destination** for 60 seconds. Go tracks TCP evidence by
  host and UDP evidence by host and port. A matching reply clears the member's
  evidence for that destination; other destinations keep their order. If every
  proxy is affected, the proxies retain their relative order.
- The displayed order keeps built-ins such as `DIRECT` and `REJECT` in their ordinary
  slots. Go reserves non-leading built-ins for after the real TCP candidates. It
  staggers at most three concurrent dials and can reach later candidates within the
  caller's deadline. UDP tries eligible members sequentially. When a real backup
  wins over a pending dial, the next connection to the same host can prefer the backup.
- Group-wide demotion requires three distinct strong-failure hosts in 60 seconds,
  or failure of the configured health URL while a peer succeeds. UDP silence,
  ordinary EOF with a pending request, canceled earlier dials and idle periods are weak
  signals. Strong and weak checks coalesce with minimum gaps of 3 and 15 seconds.
  Instance-owned cooldowns grow through 30, 60, 120, 240 and 300 seconds. In Go,
  failures on several UDP ports of one host count as one host toward demotion.
  The Go group still records individual destination faults for a member in cooldown
  and keeps the original cooldown deadline.
- Destinations sharing one local UDP endpoint use separate fallback upstream
  associations. If A keeps receiving replies while B goes silent, A stays on
  its healthy upstream and B can recover on another member independently.
  A reply clears only the exchange for an unambiguously matched endpoint.
  When Go rebuilds a QUIC association, continuation packets on the same UDP
  tuple reuse the previously accepted hostname. The matcher runs again and
  selects the same fallback group under unchanged routing policy. A new QUIC
  Initial goes through normal hostname sniffing, even on the same tuple.
- Idle alone triggers one weak health observation per continuous idle interval.
  Once the idle window expires, cleanup can retire the session if its **serving
  member** is confirmed failed or demoted and a two-second sample shows no
  progress. The fallback policy preserves healthy idle sessions through recovery
  and preference changes, including manual selection. CMA Android's explicit
  `PatchSelector` action separately closes connections whose chain includes the
  selected group. In Go, a later provider health check can retire an
  already-idle TCP stream when its serving member fails the group's health URL
  and an eligible peer for that host is healthy on the same URL. Group cleanup of
  a stalled TCP upload requires a full quiet window, zero writes in flight, and
  unchanged progress across the two-second sample.
- An established TCP stream on a failed path needs the application to reconnect.
  SimpleXray can replay a first flight of at most 256 KiB before the first reply.
  TCP reports bytes rather than application messages: if a response starts and
  then stops, silence alone looks like a healthy idle connection.
- The group-wide indicator describes the default candidate. A destination-specific
  route can already be using a backup while the indicator shows the primary.
