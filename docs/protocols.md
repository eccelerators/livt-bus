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
