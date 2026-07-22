# Changelog

## Unreleased

## 0.3.0 - 2026-07-22

- Add all six directed bridges among AXI4-Lite, non-pipelined Avalon-MM, and
  Wishbone Classic under `Livt.Bus.Bridges`.
- Preserve byte addresses and byte enables across bridge boundaries and add
  configurable downstream timeouts.
- Map AXI4-Lite and Avalon responses directly, map Wishbone timeout to
  `SLVERR`, and expose downstream status on ACK-only Wishbone-source bridges.
- Add end-to-end simulations for delayed handshakes, masked writes, read-back,
  response propagation, and timeout handling through every bridge direction.
- Keep timed-out AXI4-Lite and Avalon requests protocol-compliant until their
  late completion is drained, and expose recovery state through
  `IsRecovering()`.
- Expose `LastTimedOut()` on every bridge and verify reuse after timeout,
  including independent AXI `AW`, `W`, `B`, `AR`, and `R` stalls.
- Make configured master and bridge timeouts count exactly the requested number
  of consecutive stalled evaluations, with adjacent failure/success boundary
  regressions for every reusable master.
- Keep reusable and testing AXI4-Lite/Avalon masters protocol-compliant after a
  local timeout by draining the outstanding transaction before returning.
- Add normalized `LastResponse()` status to `IMemoryMappedMaster32`, including
  Wishbone `OKAY` and timeout mappings.
- Make the assertion-oriented common interface inherit the non-asserting
  transaction API, and enforce aligned addresses in full-word helpers.
- Correct AXI test-target channel backpressure and Wishbone cycle cancellation.

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
