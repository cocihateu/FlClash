# Mihomo fallback policy

ClashMetaForAndroid (CMA) owns the canonical Go implementation in
`core/src/foss/golang/clash/adapter/outboundgroup/fallback.go`. FlClash carries the
same observation and failure-policy code in `core/Clash.Meta/adapter/outboundgroup/fallback.go`.
Maintain both ports together; preserve FlClash's upstream-specific `UsePosition`
constructor argument and adapt each patch's hunk offsets to its pinned source.
SimpleXray implements the equivalent policy through its existing
Kotlin `FallbackRelayServer`, `GroupRuntimeController`, and `FallbackPolicy`.

`enhance-fallback.patch` contains production changes to
`adapter/outboundgroup/fallback.go`. Build Fork and the manual `build` workflow
apply it with `--include=adapter/outboundgroup/fallback.go`; the manual workflow
compiles the package remotely with `go build ./adapter/outboundgroup/...`.

## Shared contract

- Each payload write has its own RTT-based 4–10 second deadline. Completed TCP
  writes advance the response deadline. Incoming bytes complete the matching
  exchange and refresh idle observation. Outstanding responses and idle flows
  remain observable after expiry; the next write starts a fresh response window.
- A real TCP response stall, early EOF with a pending request, strong I/O error, or UDP exchange that
  goes silent moves that member behind alternatives **for the affected host** for
  60 seconds. Other hosts keep their normal order. A matching reply clears the
  host's evidence. A group with every member affected retains its normal order.
- Group-wide demotion requires three distinct strong-failure hosts in 60 seconds,
  or failure of the configured health URL while a peer succeeds. UDP silence,
  pre-response EOF, canceled earlier dials and downstream idle periods are weak
  signals. Strong and weak checks coalesce with minimum gaps of 3 and 15 seconds.
  Instance-owned cooldowns grow through 30, 60, 120, 240 and 300 seconds.
- Shared UDP transports observe pending destinations separately. Replies from an
  active destination keep its association useful while another exchange is silent.
- Cleanup samples a stalled transport over two seconds. Active transfers and
  healthy idle sessions keep their connections. Confirmed health failure can
  retire a silent download. An established TCP stream reconnects through its
  application; SimpleXray can replay up to 256 KiB before its first response.
- The group-wide indicator describes the default candidate. A destination-specific
  route can already be using a backup while the indicator shows the primary.
