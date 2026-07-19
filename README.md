# Livt.Bus

`Livt.Bus` provides reusable 32-bit memory-mapped bus masters, signal
contracts, and verification components for Livt designs. Applications can use
the masters as normal components, while tests can drive the same bus contracts
through source-level function calls.

The 0.2.0 package supports three commonly used register buses:

- `Livt.Bus.Axi4Lite.Axi4LiteMaster`: reusable AXI4-Lite transaction master.
- `Livt.Bus.Avalon.AvalonMaster`: reusable non-pipelined Avalon-MM master.
- `Livt.Bus.Wishbone.WishboneMaster`: reusable Wishbone Classic master.
- `Livt.Bus.IMemoryMappedMaster32`: common non-asserting transaction API.
- `IAxi4Lite32Master`, `IAvalon32Master`, and `IWishbone32Master`: directed
  signal contracts relative to a master.
- `Axi4LiteTestMaster`, `AvalonTestMaster`, and `WishboneTestMaster`:
  assertion-oriented masters with convenient `Write*`, `Read*`, and `Verify*`
  operations.
- Configurable AXI4-Lite, Avalon, and Wishbone target models in each protocol's
  `Testing` namespace.

## 📦 Package

```toml
[dependencies]
Livt.Bus = "0.2.0"
```

`Livt.Bus` is part of the official Livt base-library package set. Depend on it
directly when a component exposes one of the supported bus contracts or needs
to initiate memory-mapped transactions.

## 📚 Namespaces

| API | Namespace | Purpose |
|---|---|---|
| `IMemoryMappedMaster32` | `Livt.Bus` | Common timeout, read, and write transaction API |
| `IMemoryMappedBus32` | `Livt.Bus` | Compatibility test API with read/write/verify helpers |
| `Axi4LiteMaster` | `Livt.Bus.Axi4Lite` | AXI4-Lite transaction engine |
| `AvalonMaster` | `Livt.Bus.Avalon` | Non-pipelined Avalon-MM transaction engine |
| `WishboneMaster` | `Livt.Bus.Wishbone` | Wishbone Classic single-transfer engine |
| `Axi4LiteTestMaster` | `Livt.Bus.Axi4Lite.Testing` | Assertion-oriented AXI4-Lite verification API |
| `AvalonTestMaster` | `Livt.Bus.Avalon.Testing` | Assertion-oriented Avalon-MM verification API |
| `WishboneTestMaster` | `Livt.Bus.Wishbone.Testing` | Assertion-oriented Wishbone verification API |
| `*TestSlave` | Protocol `Testing` namespace | Configurable 16-word target model |

## 🔌 Common Transaction API

All reusable masters implement `IMemoryMappedMaster32`:

- `TryWrite32(address, value, byteEnable)`
- `TryRead32(address)`
- `LastReadValue()`
- `SetTimeoutTicks(timeoutTicks)` and `GetTimeoutTicks()`
- `LastTimedOut()`

Transactions are non-asserting. A return value of `true` means the protocol
completed its handshake. AXI4-Lite and Avalon expose `LastResponse()` as an
additional protocol status. The Wishbone contract in this release is the
ACK-based Classic subset and therefore reports completion and timeout only.

```livt
using Livt.Bus.Axi4Lite

component RegisterWriter
{
    master: Axi4LiteMaster
	registers: RegisterBlock

    new()
    {
        this.master = new Axi4LiteMaster()
		this.registers = new RegisterBlock(this.master)
    }

    public fn WriteControl(value: uint) bool
    {
        return this.master.TryWrite32(0x04 as uint, value, 0b1111)
    }
}
```

Most designs connect a master directly to a component constructor that accepts
the flipped protocol interface:

```livt
this.master = new Axi4LiteMaster()
this.registers = new RegisterBlock(this.master)
```

## 🧪 Verification API

Each protocol provides a test master in its `Testing` namespace for concise
register verification. Test masters offer the same non-asserting transaction
operations as their reusable counterpart alongside these assertion-oriented
helpers:

- `Write8`, `Write16`, `Write32`, and `WriteMasked32`
- `Read8`, `Read16`, and `Read32`
- `Verify8`, `Verify16`, `Verify32`, and `VerifyMasked32`

```livt
using Livt.Bus.Axi4Lite.Testing

this.master = new Axi4LiteTestMaster()
this.master.Write32(0x04 as uint, 0x12345678 as uint)
this.master.Verify32(0x04 as uint, 0x12345678 as uint)
```

Reusable components should use `Axi4LiteMaster`, `AvalonMaster`, or
`WishboneMaster`. Tests may use those same masters with explicit assertions or
the corresponding test master when the convenience operations make the test
clearer.

## ⏱️ Context-Derived Timeouts

Each master derives its default timeout from its component context:

```livt
var timeoutTicks = this.context.TicksFor(10us)
this.master.SetTimeoutTicks(timeoutTicks)
```

Assigning the master to another context automatically gives duration-based
timeouts the correct tick count for that context.

## 🧪 Build and Test

Run the complete package simulation suite:

```bash
livt test
```

To force clean regeneration without removing dependencies:

```bash
rm -rf out .livt/src.json .livt/ghdl
livt test
```

The suite covers delayed handshakes, independent AXI write channels, masked
writes, read-back, response propagation, repeated transactions, idle cleanup,
and timeout behavior across all three protocols.

Short examples live in [`docs/usage.md`](docs/usage.md). Supported protocol
profiles and integration constraints are documented in
[`docs/protocols.md`](docs/protocols.md).

## 🛠️ Development Notes

- Keep protocol-independent contracts in `namespace Livt.Bus`.
- Keep protocol APIs in namespaces such as `Livt.Bus.Axi4Lite`.
- Put assertion-oriented masters and reusable target fixtures in the
  protocol's `Testing` namespace.
- Treat all public addresses as byte addresses.
- Add functional simulations for every new handshake, response, strobe, or
  timeout behavior.

## 🚧 Outlook

Future additions may include APB, optional Wishbone error/retry signaling,
pipelined Avalon reads, configurable widths, and reusable register-bank
components.

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
