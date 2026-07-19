# Livt.Bus Usage

## AXI4-Lite

```livt
using Livt.Bus.Axi4Lite

component AxiRegisterClient
{
    master: Axi4LiteMaster
    registers: RegisterBlock

    new()
    {
        this.master = new Axi4LiteMaster()
        this.registers = new RegisterBlock(this.master)
    }

    public fn WriteAndRead(address: uint, value: uint) bool
    {
        if (this.master.TryWrite32(address, value, 0b1111) == false) {
            return false
        }
        if (this.master.LastResponse() != Axi4LiteMaster.RESPONSE_OKAY) {
            return false
        }
        return this.master.TryRead32(address)
    }
}
```

AXI write-address and write-data handshakes are independent. The transaction
completes after the write response is accepted.

## Avalon-MM

```livt
using Livt.Bus.Avalon

this.master = new AvalonMaster()
this.registers = new AvalonRegisterBlock(this.master)

var completed = this.master.TryWrite32(
    0x08 as uint,
    0xAABBCCDD as uint,
    0b0011
)
assert completed
assert this.master.LastResponse() == AvalonMaster.RESPONSE_OKAY
```

The Avalon master supports the non-pipelined profile represented by
`IAvalon32Master`. Read data and response are sampled when `waitrequest` is
deasserted.

## Wishbone Classic

```livt
using Livt.Bus.Wishbone

this.master = new WishboneMaster()
this.registers = new WishboneRegisterBlock(this.master)

assert this.master.TryWrite32(0x0C as uint, 0x12345678 as uint, 0b1111)
assert this.master.TryRead32(0x0C as uint)
assert this.master.LastReadValue() == (0x12345678 as uint)
```

The Wishbone master holds `cyc` and `stb` until `ack` is observed or the
configured timeout expires.

## Test Targets

Each protocol provides an assertion-oriented master and a small configurable
target model in its `Testing` namespace:

- `Livt.Bus.Axi4Lite.Testing.Axi4LiteTestMaster`
- `Livt.Bus.Axi4Lite.Testing.Axi4LiteTestSlave`
- `Livt.Bus.Avalon.Testing.AvalonTestMaster`
- `Livt.Bus.Avalon.Testing.AvalonTestSlave`
- `Livt.Bus.Wishbone.Testing.WishboneTestMaster`
- `Livt.Bus.Wishbone.Testing.WishboneTestSlave`

Test masters add `Write*`, `Read*`, and `Verify*` operations that assert normal
completion. Test slaves provide 16 words of memory and configurable handshake
delays. These components are intended for protocol and component tests, not
application storage.
