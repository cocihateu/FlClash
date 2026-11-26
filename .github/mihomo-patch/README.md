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

- Arm response observation before payload writes and keep it active throughout
  the connection. Each outstanding response interval has a fixed RTT-based
  4–10 second deadline; further writes retain that deadline. A response ends the
  interval, and the next payload write arms another. A write still in progress
  retains deadline protection until its completion.
- Downstream progress also starts one weak idle observation. Further received
  bytes refresh it; the next write replaces it with the fixed response interval.
  This covers stalled downloads while healthy idle sessions remain open after
  successful comparative checks.
- TCP response expiry and strong I/O failures supply strong evidence, including
  failures after the first response. Orderly EOF after establishment is normal;
  pre-response EOF and UDP silence supply weak evidence for comparative checks.
  Local SOCKS failures remain local.
- Earlier dial candidates canceled after a later candidate succeeds trigger a
  weak comparative check. UDP idle read expiry is normal; an outstanding UDP
  response timeout remains weak evidence, attributed to its destination address.
- Three distinct destination hosts with strong failures within 60 seconds can
  demote a node even while its health URL responds. Selective failures affecting
  one or two hosts still require comparative evidence: the current node fails
  while a peer succeeds. Strong and weak checks use minimum gaps of 3 and
  15 seconds. Successful traffic clears only that destination's failure evidence.
- Failure state belongs to each proxy instance. Cooldown grows through 30, 60,
  120, 240, and 300 seconds; late replies preserve an active cooldown.
- Close blocked writes at the response deadline. Original stalled flows
  reevaluate selection after asynchronous health checks. SimpleXray permits
  bounded replay before the first response; established sessions reconnect
  through the application.
- Demotion samples received bytes over 2 seconds and closes stalled connections;
  downloads making byte progress keep their connections. Observed failover time
  includes applicable check gaps, health checks, and cleanup as well as the
  response window.
