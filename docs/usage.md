# Configurable bus API

Import the protocol namespace for transaction engines, and its `.Testing`
namespace for assertion helpers and simulation targets. Width parameters default
to 32-bit addresses and 32-bit data:

```livt
using Livt.Bus.Avalon.Testing

@Test
component Example {
    master: AvalonTestMaster<7,128>
    target: AvalonTestSlave<7,128>
    new() {
        this.master = new AvalonTestMaster<7,128>()
        this.target = new AvalonTestSlave<7,128>(this.master)
    }
    @Test fn UpperByte() {
        master.WriteWord(0 as Logic[7], 0 as Logic[128])
        master.Write8(15 as Logic[7], 0xA5 as byte)
        master.VerifyWord(0 as Logic[7], (0xA5 as Logic[128]) << 120)
    }
}
```

Use `AvalonTestMaster` without arguments for the defaults. Wishbone and AXI4-Lite
use the same native word methods. `WriteMaskedWord` takes a Logic vector with one
bit per byte; `VerifyMaskedWord` takes a full-width data mask. Addresses are byte
addresses and native word operations require natural alignment.

AXI4-Lite permits 32/64-bit data by default. The explicit third argument enables
the nonstandard 128-bit profile: `Axi4LiteTestMaster<40,128,true>`.
Its parameter name is `AllowNonStandardDataWidth`.

## Fixed-width compatibility examples

The following examples use the retained uint-based API. Existing callers should
add the `32` suffix to the former component names. New configurable components
use Logic vectors and the word methods shown above. Bridges still use the fixed
32-bit signal interfaces.

# Livt.Bus Usage

## AXI4-Lite

```livt
using Livt.Bus.Axi4Lite

component AxiRegisterClient
{
    master: Axi4LiteMaster32
    registers: RegisterBlock

    new()
    {
        this.master = new Axi4LiteMaster32()
        this.registers = new RegisterBlock(this.master)
    }

    public fn WriteAndRead(address: uint, value: uint) bool
    {
		if (master.TryWrite32(address, value, 0b1111) == false) {
            return false
        }
		if (master.LastResponse() != Axi4LiteMaster32.RESPONSE_OKAY) {
            return false
        }
		return master.TryRead32(address)
    }
}
```

AXI write-address and write-data handshakes are independent. The transaction
completes after the write response is accepted.

## Avalon-MM

```livt
using Livt.Bus.Avalon

this.master = new AvalonMaster32()
this.registers = new AvalonRegisterBlock(this.master)

var completed = master.TryWrite32(
    0x08 as uint,
    0xAABBCCDD as uint,
    0b0011
)
assert completed
assert master.LastResponse() == AvalonMaster32.RESPONSE_OKAY
```

The Avalon master supports the non-pipelined profile represented by
`IAvalon32Master`. Read data and response are sampled when `waitrequest` is
deasserted.

## Wishbone Classic

```livt
using Livt.Bus.Wishbone

this.master = new WishboneMaster32()
this.registers = new WishboneRegisterBlock(this.master)

assert master.TryWrite32(0x0C as uint, 0x12345678 as uint, 0b1111)
assert master.TryRead32(0x0C as uint)
assert master.LastReadValue() == (0x12345678 as uint)
```

The Wishbone master holds `cyc` and `stb` until `ack` is observed or the
configured timeout expires.

## Protocol Bridges

Bridge constructors accept the upstream master interface and expose the
downstream master interface. For example, this connects an AXI4-Lite master to
an Avalon-MM register block:

```livt
using Livt.Bus.Axi4Lite
using Livt.Bus.Bridges

this.master = new Axi4LiteMaster32()
this.bridge = new Axi4LiteToAvalonBridge32(this.master)
this.registers = new AvalonRegisterBlock(this.bridge)

var timeoutTicks = this.context.TicksFor(20us)
bridge.SetTimeoutTicks(timeoutTicks)
```

The same constructor pattern applies to all six `*To*Bridge32` components.
The protocol named before `To` is upstream; the protocol after `To` is the
interface exposed to the target.

Every bridge exposes `LastTimedOut()`. Bridges targeting AXI4-Lite or Avalon
also expose `IsRecovering()`. When it is true, the caller has received its
timeout result but the bridge is still completing and discarding the original
downstream request:

```livt
if (bridge.LastTimedOut()) {
	while (bridge.IsRecovering()) {
		Simulation.Wait(1)
	}
}
```

Most hardware should simply leave backpressure connected and let the bridge
stall the next request. Polling is mainly useful in tests and diagnostics.

Wishbone-source bridges cannot signal a downstream error on the ACK-only
Wishbone interface. They acknowledge a completed transfer and retain status
for diagnostics:

```livt
assert wishboneMaster.TryRead32(0x10 as uint)
assert bridge.LastTimedOut() == false
assert bridge.LastResponse() == 0b00
```

## Test Targets

Each protocol provides an assertion-oriented master and a small configurable
target model in its `Testing` namespace:

- `Livt.Bus.Axi4Lite.Testing.Axi4LiteTestMaster32`
- `Livt.Bus.Axi4Lite.Testing.Axi4LiteTestSlave32`
- `Livt.Bus.Avalon.Testing.AvalonTestMaster32`
- `Livt.Bus.Avalon.Testing.AvalonTestSlave32`
- `Livt.Bus.Wishbone.Testing.WishboneTestMaster32`
- `Livt.Bus.Wishbone.Testing.WishboneTestSlave32`

Test masters add `Write*`, `Read*`, and `Verify*` operations that assert normal
completion. Test slaves provide 16 words of memory and configurable handshake
delays. These components are intended for protocol and component tests, not
application storage.

Test masters also implement `IMemoryMappedMaster32`, so generic transaction
helpers can use the same interface with reusable and assertion-oriented
masters. Full-word assertion helpers require four-byte-aligned addresses;
byte and halfword helpers align and select lanes automatically.
