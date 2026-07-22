# Supported Bus Profiles

`Livt.Bus` deliberately exposes small, explicit protocol profiles. Components
that require optional channels or pipelining should define a separate contract
instead of changing the meaning of an existing one.

## AXI4-Lite 32

`IAxi4Lite32Master` contains the five standard AXI4-Lite channels with 32-bit
addresses and data. The master handles write address and write data
independently, waits for write responses, and applies `wstrb` byte enables.

Transactions use byte addresses. Protection fields are driven as `0b000`.

## Avalon-MM 32

`IAvalon32Master` is the non-pipelined register-access profile used by Livt and
HxS integrations:

- 32-bit byte address and write data
- four byte-enable bits
- `read`, `write`, and `waitrequest` handshake
- two-bit response
- read data available when the request is accepted

This profile does not include optional burst, pending-read, or separate
read-data-valid signals.

## Wishbone Classic 32

`IWishbone32Master` is an ACK-based Wishbone Classic single-transfer profile:

- 32-bit byte address and data
- four byte-select bits
- `cyc`, `stb`, `we`, and `ack` handshake

Optional `ERR`, `RTY`, lock, tag, and burst signals are not part of this
version. A transfer that never receives `ack` ends through the master's local
timeout.

## Bridge Profile

The bridge components convert one transaction at a time in a single clock
domain. Addresses remain 32-bit byte addresses and byte enables map bit for
bit. AXI4-Lite write address and data may arrive independently; an AXI-source
bridge captures both before issuing the downstream write.

AXI4-Lite and Avalon two-bit responses map directly in either direction.
Wishbone-source bridges acknowledge completed AXI4-Lite or Avalon transactions
because the current Wishbone profile has no `ERR` output. Call
`LastResponse()` to inspect the downstream response and `LastTimedOut()` to
distinguish a timeout. On a downstream timeout no Wishbone `ack` is generated,
allowing the upstream Wishbone master to apply its own timeout policy.

AXI4-Lite- or Avalon-source bridges map a downstream Wishbone timeout to
`SLVERR` (`0b10`). A normal Wishbone acknowledgement maps to `OKAY` (`0b00`).
All bridges expose `LastTimedOut()`.

AXI4-Lite and Avalon requests remain asserted after a local timeout until the
downstream handshake and any response complete. Bridges targeting those
protocols expose `IsRecovering()` and accept no new work while it is true. This
quarantines late completions so they cannot satisfy a later transaction.
Wishbone-target bridges abort a timed-out cycle by lowering `cyc` and `stb`.

## Byte Enables

`TryWrite32` always receives a four-bit byte-enable value. Bit zero selects the
least-significant byte and bit three selects the most-significant byte.

```text
0b0001  byte 0
0b0011  lower halfword
0b1100  upper halfword
0b1111  complete word
```

Use aligned addresses for full-word transactions. Narrow writes may use an
aligned address with the appropriate data position and byte-enable pattern.

## Master Timeout Recovery

Wishbone masters abort a timed-out transfer by lowering `cyc` and `stb`.
AXI4-Lite and Avalon cannot cancel an offered request. Their `TryWrite32()` and
`TryRead32()` methods mark the transaction as timed out at the configured
deadline, continue holding the required signals until the late transaction is
safely drained, and then return `false`. A target that never completes can
therefore keep the call blocked; reset is the only protocol-safe escape.
