# Changelog

All notable changes to this package are documented here.

## 2.0.0

### Added

- Configurable `AvalonMaster`, `WishboneMaster` and `Axi4LiteMaster` components
  with compile-time address and data widths.
- Generic `IMemoryMappedMaster<Address, Data, ByteEnable>` and
  `IMemoryMappedBus<Address, Data, ByteEnable>` transaction contracts.
- Protocol-specific `IAvalonMaster`, `IWishboneMaster` and `IAxi4LiteMaster`
  signal contracts with exact address, data and byte-enable types.
- Configurable test masters and test slaves for Avalon-MM, Wishbone Classic and
  AXI4-Lite, including native-word, masked-word and byte-lane operations.
- 32-, 64- and 128-bit protocol tests covering upper address/data bits, byte
  lanes, masks, delayed responses, error propagation and timeout recovery.
- Explicit alignment control for test masters. Avalon and Wishbone additionally
  support masked reads for target-side byte-enable testing.

### Changed

- The short master and test-component names now refer to configurable APIs using
  `Logic[AddressWidth]` and `Logic[DataWidth]` values without `uint` truncation.
- Former fixed-width components and interfaces are retained under explicit
  `…32` names. The six protocol bridges remain fixed-width `…Bridge32`
  components.
- Wishbone request sequencing guarantees one accepted transfer for one request,
  including with an always-ready target.
- Avalon test targets retain an accepted response until the requester releases
  the transaction, preventing repeated acceptance across clock edges.
- Timeout tests now cover expiry immediately before and at handshake boundaries.

### Breaking changes and migration

- Rename uses of the former `Axi4LiteMaster`, `AvalonMaster`, `WishboneMaster`
  and their test-master/test-slave counterparts with the `32` suffix when the
  existing uint-based API and fixed-width interfaces must be preserved.
- New code should use the short configurable names and `WriteWord`,
  `WriteMaskedWord`, `ReadWord`, `VerifyWord` and `VerifyMaskedWord` operations.
- The earlier configurable development names prefixed with `Generic` are now the
  short public names.
- The AXI4-Lite opt-in parameter for a nonstandard 128-bit data path is named
  `AllowNonStandardDataWidth`.
- Data widths are 32, 64 or 128 bits. AXI4-Lite accepts 32 and 64 bits by
  default; 128 bits requires the explicit nonstandard-width opt-in.

### Validation

- Package tests cover all three protocols at 32, 64 and 128 bits, fixed-width
  compatibility APIs, always-ready targets, timeout boundaries and every
  supported fixed-width bridge direction.
- The package does not claim clock-domain crossing, width conversion, burst or
  pipelined operation, or hardware timing/resource qualification.

## Earlier releases

Earlier release history remains available in Git.
