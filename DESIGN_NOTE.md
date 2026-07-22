# Bridge Design Note

## Decision

`Livt.Bus` provides six pair-specific bridge components instead of routing all
protocols through a stored generic transaction interface.

The pair-specific design keeps signal timing explicit in Livt v0.0.10, avoids
additional protocol-facing API changes, and lets AXI4-Lite's independent write
channels be handled at their actual handshake boundary. All bridges use a
single-outstanding state machine and direct signal assignments.

## Compatibility Contract

- Existing protocol signal interfaces remain unchanged.
- `IMemoryMappedMaster32.LastResponse()` is an additive normalized status API.
- All bridge APIs are additive under `Livt.Bus.Bridges`.
- Public addresses remain byte addresses and byte enables retain their bit
  positions.
- Default downstream timeout is derived from the component context.
- Timeout values count exact consecutive stalled evaluations.
- AXI4-Lite and Avalon requests remain protocol-compliant after a local timeout;
  late completions are quarantined until recovery finishes.
- The ACK-only Wishbone profile cannot carry AXI4-Lite/Avalon error status;
  Wishbone-source bridges expose `LastResponse()` and `LastTimedOut()` instead.

Reusable and testing masters follow the same recovery rule. Because their
transaction functions directly own the protocol outputs, AXI4-Lite and Avalon
calls return a timeout result only after the late transaction has drained.
Wishbone calls can return immediately after aborting the active cycle.

## Recovery Contract

AXI4-Lite `VALID` signals remain asserted until their matching `READY`
handshake, and accepted writes/reads are drained through `B`/`R`. Avalon
requests and payloads remain stable while `waitrequest` is high. Components
targeting either protocol expose `IsRecovering()` and apply backpressure until
the timed-out downstream transaction is fully discarded. Wishbone-target
bridges abort by lowering `cyc` and `stb`, as permitted by the selected Classic
profile.

## Deliberate Non-Goals

This bridge set does not implement clock-domain crossing, burst conversion,
multiple outstanding transactions, data-width conversion, address mapping, or
optional Wishbone `ERR`/`RTY` signaling. Those features require separate,
explicit interfaces rather than silently changing the current profiles.
