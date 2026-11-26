# Mihomo fallback policy

ClashMetaForAndroid (CMA) owns the canonical Go implementation in
`core/src/foss/golang/clash/adapter/outboundgroup/fallback.go`. FlClash carries the
same observation and failure-policy code in `core/Clash.Meta/adapter/outboundgroup/fallback.go`.
Maintain both ports together; preserve FlClash's upstream-specific `UsePosition`
constructor argument and adapt each patch's hunk offsets to its pinned source.
SimpleXray implements the equivalent policy through its existing
Kotlin `FallbackRelayServer`, `GroupRuntimeController`, and `FallbackPolicy`.

Apply both patches to the pinned core in the order shown in the [build instructions](../../.agents/commands.md#building).
`enhance-fallback.patch` updates the group policy; its `--include=adapter/outboundgroup/fallback.go`
filter applies to that patch only. Apply `fallback-udp.patch` in full for destination-specific UDP routing.
The manual `build` workflow compiles the group package with `go build ./adapter/outboundgroup/...`.

## Shared contract

- Each TCP write has its own RTT-based 4–10 second window to complete. For an
  unanswered exchange, completed writes can extend the response deadline only
  to two windows from the first unanswered write. Further accepted writes leave
  the exchange stalled; a real downstream reply clears it.
- If unsolicited data arrives after an established TCP stream's idle window
  expires, the first write within one window of that data counts as a likely
  reply rather than a new request. Its blocked-write deadline still applies.
  This is a timing heuristic, independent of the payload protocol.
- A TCP response stall, ordinary EOF with a request outstanding (even after an
  earlier reply), a strong I/O error, or UDP silence can change the next route
  **for the affected host** for 60 seconds. Host ranking reorders only proxy
  slots; built-ins such as `DIRECT` and `REJECT` keep their positions in the
  group's ordinary order. Other hosts keep their order. A matching reply clears
  the member's evidence for that host. If every proxy is affected, their order holds.
- Group-wide demotion requires three distinct strong-failure hosts in 60 seconds,
  or failure of the configured health URL while a peer succeeds. UDP silence,
  ordinary EOF with a pending request, canceled earlier dials and idle periods are weak
  signals. Strong and weak checks coalesce with minimum gaps of 3 and 15 seconds.
  Instance-owned cooldowns grow through 30, 60, 120, 240 and 300 seconds. In Go,
  a member in cooldown still records individual host faults; the existing
  group cooldown keeps its deadline.
- Destinations sharing one local UDP endpoint use separate fallback upstream
  associations. If A keeps receiving replies while B goes silent, A stays on
  its healthy upstream and B can recover on another member independently.
  A reply clears only the exchange for an unambiguously matched endpoint.
- Idle alone triggers one weak health observation per continuous idle interval.
  Once the idle window expires, cleanup can retire the session if its **serving
  member** is confirmed failed or demoted and a two-second sample shows no
  progress. Ordinary preference changes, recovery and manual selection preserve
  healthy idle sessions. Go cleanup follows nested wrappers to find the
  observation for the serving member. Transfers making progress stay connected.
- An established TCP stream on a failed path needs the application to reconnect.
  SimpleXray can replay a first flight of at most 256 KiB before the first reply.
- The group-wide indicator describes the default candidate. A destination-specific
  route can already be using a backup while the indicator shows the primary.
