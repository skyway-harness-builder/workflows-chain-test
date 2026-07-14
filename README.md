# workflows-chain-test

A tiny public workflow collection whose only job is to prove how nested skyway
workflows behave under the library resolver and its `enabled` registry gate.
Every workflow is pure bash, costs nothing, needs no credentials, and only
appends a line to `/tmp/skyway-chain-test.log`, so it is safe to run repeatedly
against a live daemon as a verification fixture.

## The two nesting mechanisms

**Invoke chain (naming crosses the gate).** `chain-invoke-root` runs a bash node,
then an invoke node calls `chain-invoke-leaf` by name. An explicit invoke is
composition by naming, so it always reaches the target regardless of the
target's `enabled` flag.

**Event chain (matching respects the gate).** `chain-event-emitter` emits
`chaintest.ping`. `chain-event-listener` declares `trigger.sky_event.event =
"chaintest.ping"` and re-emits `chaintest.pong`, which `chain-event-final`
listens for. Automatic trigger matching only fires a listener when that listener
is registered `enabled=1`. With the listeners gated off, firing the emitter
produces exactly one log line and the cascade stops.

## Proof procedure (once the library resolver tier ships)

Sync this repo into the library cache, then:

```
: > /tmp/skyway-chain-test.log

# 1. Invoke chain crosses the gate even at enabled=0
skyway run chain-invoke-root
#   log: invoke-root:start  +  invoke-leaf:land marker=from-root

# 2. Event chain is gated: listeners at enabled=0 do not fire
skyway run chain-event-emitter
#   log: event-emitter:fire   (and nothing else)

# 3. Enable the listeners, then the cascade completes
#   (enable chain-event-listener and chain-event-final in the library registry)
skyway run chain-event-emitter
#   log: event-emitter:fire  ->  event-listener:relay stage=1  ->  event-final:land stage=2
```

## Workflows

| workflow | trigger | role |
| --- | --- | --- |
| `chain-invoke-root` | manual | logs, then invokes the leaf by name |
| `chain-invoke-leaf` | manual | logs the marker it received |
| `chain-event-emitter` | manual | logs, emits `chaintest.ping` |
| `chain-event-listener` | `sky_event chaintest.ping` | logs stage, emits `chaintest.pong` |
| `chain-event-final` | `sky_event chaintest.pong` | logs the final stage |
