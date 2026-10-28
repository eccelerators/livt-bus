# Livt.Bus

`Livt.Bus` provides configurable memory-mapped bus masters, signal
contracts, protocol bridges, and test components for Livt designs.
Applications can use the masters and bridges as normal components, while tests
can drive the same bus contracts through source-level function calls.

The configurable API supports three register buses:

- `AvalonMaster`, `WishboneMaster`, and `Axi4LiteMaster`: transaction engines.
- `AvalonTestMaster`, `WishboneTestMaster`, and `Axi4LiteTestMaster`: assertion helpers.
- Matching `…TestSlave` components: configurable simulation targets.
- `IMemoryMappedMaster<Address,Data,ByteEnable>` and `IMemoryMappedBus<Address,Data,ByteEnable>`: shared method contracts.
- `IAvalonMaster<Address,Data,ByteEnable>`, `IWishboneMaster<Address,Data,ByteEnable>` and `IAxi4LiteMaster<Address,Data,ByteEnable>`: directed signal contracts.
- Explicit `…32` components and interfaces: retained fixed-width API.
- Six `…Bridge32` components: fixed-width protocol conversion.

## Choosing the API

Use `AvalonMaster`, `WishboneMaster` or `Axi4LiteMaster` for new code. All default
to 32-bit addresses and data; supply `<AddressWidth, DataWidth>` for wider buses.
Use the corresponding `…TestMaster` in tests and `…TestSlave` for a simulated
target. The configurable `WishboneMaster` uses a single-owner clocked engine
and direct handshake wires so one acknowledged request produces one transfer,
including against always-ready targets.

`AvalonTestSlave` holds an accepted response until the request is
released, so a delegated request held across clock edges counts once. Native transaction values are Logic vectors and test operations use
`WriteWord`, `ReadWord` and `VerifyWord`.

```livt
master: Axi4LiteMaster
wideMaster: Axi4LiteMaster<40,64>
```

**Source migration:** The former fixed-width `Axi4LiteMaster`, `AvalonMaster`,
`WishboneMaster` and their `TestMaster`/`TestSlave` counterparts now have a `32`
suffix. Rename those references to retain the uint-based transaction API and
existing `I…32Master` connections. Their implementations are retained. The new
configurable components take the short names formerly used by that API; this is
a source-breaking rename, not automatic compatibility. Bridges remain 32-bit.
Earlier development names `Generic…` become `…`; the AXI opt-in parameter
`ExtendedHxS` becomes `AllowNonStandardDataWidth` (same third argument).

## Raw addresses and read lane enables

Native calls check word alignment by default. To test a target's handling of
unaligned byte addresses, explicitly disable that check:

```livt
avalon: AvalonTestMaster<24,32,false>
wishbone: WishboneTestMaster<24,32,false>
axi: Axi4LiteTestMaster<24,32,false,false>
```

The additional final generic is `CheckAlignment`. AXI's preceding generic remains
`AllowNonStandardDataWidth`. Disabling the check sends the supplied address
unchanged; it does not shift data, align the address, or split the transfer.
`TryWrite` already takes explicit byte enables. Avalon and Wishbone additionally
provide `TryReadMasked(address, byteEnable)` for read-side byte-enable testing.
AXI4-Lite has no read byte-enable signal and continues to use `TryRead`.
`TryRead` selects every native lane on Avalon/Wishbone. Return values and timeout
behavior are the same as the existing native transaction methods.

## 📦 Package

```toml
[dependencies]
Livt.Bus = "2.0.0"
```

`Livt.Bus` is part of the official Livt base-library package set. Depend on it
directly when a component exposes one of the supported bus contracts or needs
to initiate memory-mapped transactions.

See [CHANGELOG.md](CHANGELOG.md) for release notes and migration guidance.

## Fixed-width API namespaces

| API | Namespace | Purpose |
|---|---|---|
| `IMemoryMappedMaster32` | `Livt.Bus` | Common timeout, read, and write transaction API |
| `IMemoryMappedBus32` | `Livt.Bus` | Extended test API with non-asserting and read/write/verify helpers |
| `Axi4LiteMaster32` | `Livt.Bus.Axi4Lite` | AXI4-Lite transaction engine |
| `AvalonMaster32` | `Livt.Bus.Avalon` | Non-pipelined Avalon-MM transaction engine |
| `WishboneMaster32` | `Livt.Bus.Wishbone` | Wishbone Classic single-transfer engine |
| `Axi4LiteTestMaster32` | `Livt.Bus.Axi4Lite.Testing` | Assertion-oriented AXI4-Lite test API |
| `AvalonTestMaster32` | `Livt.Bus.Avalon.Testing` | Assertion-oriented Avalon-MM test API |
| `WishboneTestMaster32` | `Livt.Bus.Wishbone.Testing` | Assertion-oriented Wishbone test API |
| `*TestSlave` | Protocol `Testing` namespace | Configurable 16-word target model |
| `*To*Bridge32` | `Livt.Bus.Bridges` | Single-outstanding protocol conversion |

## 🔀 Protocol Bridges

Every directed pairing among the three supported buses is available:

| Upstream | Downstream | Component |
|---|---|---|
| AXI4-Lite | Avalon-MM | `Axi4LiteToAvalonBridge32` |
| AXI4-Lite | Wishbone | `Axi4LiteToWishboneBridge32` |
| Avalon-MM | AXI4-Lite | `AvalonToAxi4LiteBridge32` |
| Avalon-MM | Wishbone | `AvalonToWishboneBridge32` |
| Wishbone | AXI4-Lite | `WishboneToAxi4LiteBridge32` |
| Wishbone | Avalon-MM | `WishboneToAvalonBridge32` |

Bridges are same-clock, 32-bit, byte-addressed, and single-outstanding. They
preserve byte enables and apply a configurable timeout to downstream stalls.
Timed-out AXI4-Lite and Avalon requests are drained before another request is
accepted, so late responses cannot cross transaction boundaries.
They do not perform clock-domain crossing, width conversion, bursts, or
pipelining. See [`docs/bridges.md`](docs/bridges.md) for response semantics and
wiring examples.

## 🔌 Common Transaction API

All reusable masters implement `IMemoryMappedMaster32`:

- `TryWrite32(address, value, byteEnable)`
- `TryRead32(address)`
- `LastReadValue()`
- `LastResponse()`
- `SetTimeoutTicks(timeoutTicks)` and `GetTimeoutTicks()`
- `LastTimedOut()`

Transactions are non-asserting. A return value of `true` means the protocol
completed its handshake; inspect `LastResponse()` to determine the normalized
transaction status. AXI4-Lite and Avalon preserve their native response.
Wishbone maps `ack` to `OKAY` and a local timeout to `SLVERR`.

```livt
using Livt.Bus.Axi4Lite

component RegisterWriter
{
	master: Axi4LiteMaster32
	registers: RegisterBlock

	new()
	{
		this.master = new Axi4LiteMaster32()
		this.registers = new RegisterBlock(this.master)
	}

	public fn WriteControl(value: uint) bool
	{
		return master.TryWrite32(0x04 as uint, value, 0b1111)
	}
}
```

Most designs connect a master directly to a component constructor that accepts
the flipped protocol interface:

```livt
this.master = new Axi4LiteMaster32()
this.registers = new RegisterBlock(this.master)
```

## 🧪 Test API

Each protocol provides a test master in its `Testing` namespace for concise
register checks. Test masters offer the same non-asserting transaction
operations as their reusable counterpart alongside these assertion-oriented
helpers:

- `Write8`, `Write16`, `Write32`, and `WriteMasked32`
- `Read8`, `Read16`, and `Read32`
- `Verify8`, `Verify16`, `Verify32`, and `VerifyMasked32`

```livt
using Livt.Bus.Axi4Lite.Testing

this.master = new Axi4LiteTestMaster32()
master.Write32(0x04 as uint, 0x12345678 as uint)
master.Verify32(0x04 as uint, 0x12345678 as uint)
```

Reusable components should use `Axi4LiteMaster32`, `AvalonMaster32`, or
`WishboneMaster32`. Tests may use those same masters with explicit assertions or
the corresponding test master when the convenience operations make the test
clearer.

## ⏱️ Context-Derived Timeouts

Each master derives its default timeout from its component context:

```livt
var timeoutTicks = this.context.TicksFor(10us)
master.SetTimeoutTicks(timeoutTicks)
```

Assigning the master to another context automatically gives duration-based
timeouts the correct tick count for that context. Livt.Bus 1.0.1 requires the
Livt v0.0.10 compiler or newer.

## 🧪 Build and Test

Run the complete package simulation suite:

```bash
livt test
```

To force clean regeneration without removing dependencies:

```bash
livt clean
livt test
```

The suite covers delayed handshakes, independent AXI write channels, masked
writes, read-back, response propagation, repeated transactions, idle cleanup,
timeout behavior, and all six bridge directions.

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

Future additions may include APB, a Wishbone profile with error/retry
signaling, pipelined Avalon reads, clock-domain and width conversion, and
reusable register-bank components.

## Configurable native-width buses

`AvalonMaster<AddressWidth=32, DataWidth=32>`,
`WishboneMaster<AddressWidth=32, DataWidth=32>` and
`Axi4LiteMaster<AddressWidth=32, DataWidth=32, AllowNonStandardDataWidth=false>`
provide non-asserting native transactions. Each protocol's `.Testing` namespace
provides `…TestMaster` and `…TestSlave` components with the same
width parameters. Fixed-width implementations are retained under `…32` names; bridges retain their names.

Data widths are 32, 64 or 128 bits; byte enables are derived as DataWidth / 8.
Address widths range from the byte-lane index width through 64 bits. Addresses
are byte addresses. Native word operations assert natural alignment; Write8
and Read8 select a lane within the aligned native word. Values and addresses use
`Logic[DataWidth]` and `Logic[AddressWidth]`, avoiding uint truncation.
128-bit AXI4-Lite is an HxS extension, not standard AXI4-Lite: it requires the
explicit third argument `true`. The standard profile permits 32 and 64 bits.

```livt
master: AvalonTestMaster<7,128>
// In the constructor:
this.master = new AvalonTestMaster<7,128>()
// In a test method:
master.WriteWord(0 as Logic[7], (0xA5 as Logic[128]) << 120)
master.Write8(15 as Logic[7], 0xBC as byte)
master.VerifyWord(0 as Logic[7], (0xBC as Logic[128]) << 120)
```

Use `WriteWord`, `WriteMaskedWord`, `ReadWord`, `VerifyWord`,
`VerifyMaskedWord`, `Write8` and `Read8` for assertions. A Word means the native
configured width, not necessarily 32 bits. The Word suffix also avoids a VHDL
name collision with Avalon read/write pins. Each word operation uses one native
bus transaction. A zero byte-enable mask preserves every data lane.

Non-asserting code uses `TryWrite`, `TryRead`, `LastReadValue`, `LastResponse`,
`LastTimedOut`, and the timeout accessors inherited from the generic engine.
The assertion helpers require successful completion and an OK response.

Transport interfaces are type-generic: for example,
`IAvalonMaster<Logic[7], Logic[128], Logic[16]>`. Bindings borrow this using
`flip`; the component's numeric configuration derives the signal types.
`IMemoryMappedMaster<Address,Data,ByteEnable>` and
`IMemoryMappedBus<Address,Data,ByteEnable>` provide shared method contracts.
HxS bindings validate address capacity and exact data/byte-enable widths against
the original HxS interface. Only direct Avalon HxS bindings are qualified so far.

Generic test slaves are small simulation fixtures with a mirrored 16-word
window. They retain the full observed address and accepted-write count so tests
can verify address preservation and single-transfer behavior. They are not
production address decoders. Their byte merging uses a pure bounded helper;
inherited constants are locally derived for portable code generation.

Validation covers nine protocol/width combinations, including upper data/address
bits, byte lanes, masks, delayed responses, errors and timeout recovery, alongside
the fixed-width and bridge suite. The compiler and VHDL extension must support
dependent interface widths and wide HxS bindings.

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
