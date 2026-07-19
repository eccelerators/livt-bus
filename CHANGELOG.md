# Changelog

## 0.2.0

- Add reusable AXI4-Lite, Avalon-MM, and Wishbone Classic transaction masters.
- Add the protocol-neutral `IMemoryMappedMaster32` transaction contract.
- Add assertion-oriented AXI4-Lite, Avalon-MM, and Wishbone Classic testing
  masters under their protocol `Testing` namespaces.
- Add configurable Avalon-MM and Wishbone Classic target models for tests.
- Move `Axi4LiteTestMaster` to `Livt.Bus.Axi4Lite.Testing` before publication.
- Add functional simulations for delayed handshakes, masked writes, responses,
  read-back, idle outputs, and timeouts across all supported protocols.

## 0.1.0

- Add the AXI4-Lite master contract and `Axi4LiteTestMaster`.
- Add protocol signal contracts for Avalon-MM and Wishbone Classic.
- Add a configurable AXI4-Lite target model and timeout simulations.
