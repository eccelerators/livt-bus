# Protocol Bridges

The `Livt.Bus.Bridges` namespace contains direct, synthesizable adapters for
every direction between the package's AXI4-Lite, Avalon-MM, and Wishbone
Classic profiles.

## Selection

| Source master | Target interface | Bridge |
|---|---|---|
| AXI4-Lite | Avalon-MM | `Axi4LiteToAvalonBridge32` |
| AXI4-Lite | Wishbone | `Axi4LiteToWishboneBridge32` |
| Avalon-MM | AXI4-Lite | `AvalonToAxi4LiteBridge32` |
| Avalon-MM | Wishbone | `AvalonToWishboneBridge32` |
| Wishbone | AXI4-Lite | `WishboneToAxi4LiteBridge32` |
| Wishbone | Avalon-MM | `WishboneToAvalonBridge32` |

The constructor receives the source master. The bridge itself implements the
target-side master interface:

```livt
using Livt.Bus.Avalon
using Livt.Bus.Bridges

this.source = new AvalonMaster()
this.bridge = new AvalonToWishboneBridge32(this.source)
this.target = new WishbonePeripheral(this.bridge)
```

Use `SetTimeoutTicks()` and `GetTimeoutTicks()` to control the maximum wait for
each downstream handshake stage. A value of `N` permits exactly `N` consecutive
stalled evaluations; zero times out on the first stalled evaluation.

## Supported Semantics

- one transaction outstanding at a time
- one shared clock domain
- 32-bit byte addresses and data
- four byte-enable/select bits, preserved without translation
- reads and writes, with writes preferred if both are requested together
- independently accepted AXI4-Lite write-address and write-data channels

The bridges do not provide bursts, pipelining, clock-domain crossing, address
remapping, or data-width conversion.

## Responses and Timeouts

AXI4-Lite and Avalon use compatible two-bit response values, so their bridges
forward responses unchanged. A Wishbone acknowledgement maps to `OKAY`
(`0b00`). A Wishbone timeout maps to `SLVERR` (`0b10`) for AXI4-Lite or Avalon
upstreams.

`LastTimedOut()` is available on every bridge and distinguishes a locally
generated timeout from a downstream `SLVERR` response.

The package's Wishbone profile contains `ack` but no `err` or `rty`. Therefore
a Wishbone-source bridge acknowledges a downstream AXI4-Lite or Avalon error
and records it in `LastResponse()`. `LastTimedOut()` reports downstream timeout;
in that case the bridge withholds `ack` so the upstream master can time out.

AXI4-Lite and Avalon requests cannot be withdrawn after they have been offered.
After reporting a timeout, a bridge targeting either protocol keeps the request
and payload stable, accepts any outstanding response, and discards that late
completion. `IsRecovering()` reports this drain interval. The bridge stalls or
ignores new source requests until it returns `false`, preventing a late response
from being associated with a later transaction. If the target never completes,
recovery intentionally continues until reset.

Wishbone Classic cycles are abortable by deasserting `cyc` and `stb`, so bridges
targeting Wishbone need no downstream drain interval.

## Latency and Throughput

The bridges favor deterministic control-path behavior over throughput. They
allow one outstanding transaction and use these protocol phases when the target
does not stall:

| Direction | Bridge phases |
|---|---|
| AXI4-Lite to Avalon or Wishbone | capture independent AXI request channels, downstream transfer, AXI response |
| Avalon or Wishbone to AXI4-Lite | AXI address/data transfer, AXI response, source completion |
| Avalon to Wishbone or Wishbone to Avalon | downstream transfer, source completion |

Each phase crosses a clocked state boundary; target and source models may add
their own registered boundaries. This is appropriate for register and control
traffic. A datapath requiring one transfer per tick needs a pipelined bridge,
which is outside this package's single-outstanding contract.

## Choosing a Timeout

Configure the bridge timeout longer than the target's worst-case response and
shorter than the upstream master's timeout:

```livt
var bridgeTimeout = this.context.TicksFor(20us)
var sourceTimeout = this.context.TicksFor(25us)
bridge.SetTimeoutTicks(bridgeTimeout)
source.SetTimeoutTicks(sourceTimeout)
```

This ordering lets the bridge return `SLVERR` where the upstream protocol can
represent it, or release a stalled Wishbone source through its own timeout.
For AXI4-Lite and Avalon targets, also budget for the possibility that a timed-
out transaction must drain before the bridge can accept another request.
