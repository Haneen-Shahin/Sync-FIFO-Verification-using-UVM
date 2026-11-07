<div align="center">

# Synchronous FIFO Verification with SystemVerilog and UVM

**A self-checking UVM verification environment for a parameterizable synchronous FIFO, combining constrained-random stimulus, a cycle-accurate reference model, functional coverage, and SystemVerilog Assertions.**

[![SystemVerilog](https://img.shields.io/badge/SystemVerilog-IEEE%201800-blue)](https://en.wikipedia.org/wiki/SystemVerilog)
[![UVM](https://img.shields.io/badge/UVM-1.2-0070C0)](https://www.accellera.org/downloads/standards/uvm)
[![Simulator](https://img.shields.io/badge/Simulator-QuestaSim-00B0F0)](https://eda.sw.siemens.com/en-US/ic/questa/)
[![Functional Coverage](https://img.shields.io/badge/Functional%20Coverage-100%25-brightgreen)](#functional-coverage)
[![DUT Code Coverage](https://img.shields.io/badge/DUT%20Code%20Coverage-100%25-brightgreen)](#verification-results)

</div>

---

## Project Overview

This project verifies a single-clock synchronous FIFO using a complete SystemVerilog/UVM testbench. The environment applies constrained-random stimulus through five sequences and checks the DUT behavior using a cycle-accurate golden model, functional coverage, and assertion-based verification.

The verification focuses on FIFO operation under normal and boundary conditions, including reset, read and write operations, simultaneous accesses, pointer wraparound, status flags, overflow, and underflow.

## Verification Results

| Metric | Result |
|:---|:---|
| Transactions checked | **2,301** |
| Scoreboard mismatches | **0** |
| Functional coverage | **100%** — `cg_FIFO`: 68/68 bins; `cg_rst`: 4/4 bins |
| DUT code coverage | **100%** — statements, branches, conditions, and toggle |
| SystemVerilog Assertions | **15 assertions**, each with a corresponding cover directive |
| Verification plan | **27 requirements** |
| RTL issues identified and fixed | **5** |

For the detailed coverage results, see [`coverage_report.txt`](coverage_report.txt).

---

## Design Under Test

The DUT is a single-clock synchronous FIFO with an asynchronous active-low reset.

| Parameter | Configuration |
|:---|:---|
| `FIFO_WIDTH` | 16 bits |
| `FIFO_DEPTH` | 8 entries |

| Inputs | Outputs |
|:---|:---|
| `clk`, `rst_n`, `wr_en`, `rd_en`, `data_in` | `data_out`, `wr_ack`, `overflow`, `underflow` (registered) |
| | `full`, `empty`, `almostfull`, `almostempty` (combinational) |

### Key behaviors verified

- A write request while full is rejected and raises `overflow`.
- A read request while empty is rejected and raises `underflow`.
- Read and write pointers wrap from `FIFO_DEPTH - 1` back to zero.
- Simultaneous read and write operations are handled according to the FIFO state:
  - **Normal state:** both operations are accepted.
  - **Full state:** the read is accepted.
  - **Empty state:** the write is accepted.
- Status flags follow the FIFO occupancy count:
  - `empty` when count is 0.
  - `almostempty` when count is 1.
  - `almostfull` when count is 7.
  - `full` when count is 8.

---

## UVM Verification Architecture

The testbench is organized into standard UVM components. Sequences generate transactions, the driver applies stimulus to the DUT, and the monitor publishes sampled transactions to the scoreboard and coverage collector. A separate assertion module checks protocol and design properties.

<div align="center">

![UVM Testbench Architecture](UVM.png)

*UVM verification environment architecture*

</div>

### Main components

| Component | Responsibility |
|:---|:---|
| `FIFO_top` | Generates the clock, instantiates the interface and DUT, binds the assertions, configures the virtual interface through `uvm_config_db`, and starts the test. |
| `FIFO_if` | Connects the testbench to the DUT and provides access to DUT signals and internal signals used by assertions. |
| `FIFO_test` | Creates the environment and runs the five sequences in order under one objection. |
| `FIFO_config` | Distributes the virtual interface handle to the relevant UVM components. |
| `FIFO_seq_item` | Defines the transaction fields, response fields, and stimulus constraints. |
| `FIFO_sequencer` | Sends sequence items to the driver. |
| `FIFO_driver` | Drives the input signals through the virtual interface at the falling clock edge. |
| `FIFO_monitor` | Samples the inputs and outputs at the falling clock edge and broadcasts each sampled item through an analysis port. |
| `FIFO_scoreboard` | Implements the golden reference model and compares all eight DUT outputs against the expected values. |
| `FIFO_coverage` | Collects functional coverage through coverpoints and crosses. |
| `FIFO_sva` | Contains the assertions bound to the DUT. |

## Transaction Flow

1. A sequence randomizes a `FIFO_seq_item` and sends it to the sequencer.
2. The driver receives the item and applies `data_in`, `rst_n`, `wr_en`, and `rd_en` through the interface.
3. The DUT samples the inputs on the rising clock edge and updates its outputs. The bound assertions evaluate the relevant properties.
4. At the falling clock edge, the monitor samples the inputs and resulting outputs, then publishes the transaction through its analysis port.
5. The scoreboard updates its golden model and compares the expected outputs with the DUT outputs. The coverage collector samples its covergroups.
6. After all sequences complete, the test drops its objection and the scoreboard reports the matching and mismatching transaction counts.

---

## Stimulus and Test Sequences

### Constrained-random stimulus

The `FIFO_seq_item` applies the following constraints:

| Field | Constraint |
|:---|:---|
| `rst_n` | `dist {1 := 98, 0 := 2}` — allows reset to occur randomly during each sequence. |
| `wr_en` | `dist {1 := 70, 0 := 30}` — weights are defined in `shared_pkg`. |
| `rd_en` | `dist {1 := 30, 0 := 70}` — weights are defined in `shared_pkg`. |
| `data_in` | Unconstrained 16-bit random data. |

### Sequence execution

The test runs five sequences in the following order:

| # | Sequence | Items | Purpose |
|:--:|:---|---:|:---|
| 1 | `FIFO_rst_seq` | 1 | Establishes a known reset state. |
| 2 | `FIFO_wr_seq` | 100 | Exercises writing, FIFO filling, and write attempts while full. |
| 3 | `FIFO_rd_seq` | 100 | Exercises reading, FIFO draining, and read attempts while empty. |
| 4 | `FIFO_wr_rd_seq2` | 100 | Applies simultaneous read and write requests on every cycle. |
| 5 | `FIFO_wr_rd_seq1` | 2,000 | Repeats 100 alternating bursts of 10 writes followed by 10 reads to exercise occupancy changes and pointer wraparound. |

---

## Checking and Verification Strategy

### Golden-model scoreboard

The scoreboard maintains an independent reference memory, read pointer, write pointer, and occupancy count. It models reset, write-only, read-only, and simultaneous read/write behavior, including the special cases when the FIFO is full or empty.

For every monitored transaction, it compares the following eight outputs using case inequality (`!==`):

`data_out`, `wr_ack`, `overflow`, `underflow`, `full`, `empty`, `almostfull`, and `almostempty`.

The scoreboard counts matching and mismatching results. The completed run reported **2,301 matching transactions and zero mismatches**.

### SystemVerilog Assertions

The `FIFO_sva` module contains 15 assertions, each paired with a cover directive.

- **Deferred immediate assertions (`assert final`):** Used for asynchronous reset behavior and combinational flag checks. These are evaluated in an `always_comb` block at the final settled value of the time step.
- **Concurrent assertions:** Used for clocked behavior, including write acknowledgment, overflow and underflow detection, pointer wraparound, and threshold checks.

| Assertion type | Properties |
|:---|:---|
| Deferred immediate | `reset_flags`, `RST_wr_ptr`, `RST_rd_ptr`, `RST_count`, `async_reset_seq` |
| Deferred immediate | `empty_flag`, `full_flag`, `almost_full`, `almost_empty` |
| Concurrent | `write_ack`, `overflow_detection`, `underflow_detection` |
| Concurrent | `wr_wraparound`, `rd_wraparound`, `threshold` |

Assertions are enabled by compiling with `+define+SIM`.

---

## Functional Coverage

The environment uses two covergroups:

- **`cg_FIFO`:** Includes nine coverpoints: `wr_en`, `rd_en`, `full`, `almostfull`, `empty`, `almostempty`, `overflow`, `underflow`, and `wr_ack`. It also includes seven crosses: `cross_full`, `cross_almost_full`, `cross_empty`, `cross_almost_empty`, `cross_overflow`, `cross_underflow`, and `cross_wr_ack`. Impossible combinations are excluded using `ignore_bins`.
- **`cg_rst`:** Covers reset assertion, reset deassertion, and both reset edges.

The monitored flags represent the values after the clock edge. Therefore, the coverage collector crosses the applied inputs with the flags representing the state in which those inputs were applied.

**Achieved functional coverage:**
- `cg_FIFO`: 68/68 bins
- `cg_rst`: 4/4 bins
- **Overall: 100%**

---

## Verification Plan

The verification plan contains 27 requirements. Each requirement is mapped to stimulus, coverage, and a checking mechanism through the scoreboard or assertions.

| Group | Requirements | Verification focus |
|:---|:---|:---|
| Reset | FIFO_1 – FIFO_3 | Pointers, counter, sequential outputs, and flags after reset. |
| Write and read | FIFO_4 – FIFO_7 | Accepted and rejected operations, write acknowledgment, overflow, and underflow. |
| Flags | FIFO_8 – FIFO_11 | Status flags checked against the occupancy count. |
| Simultaneous access | FIFO_12 – FIFO_14 | Simultaneous operations in normal, empty, and full states. |
| Pointers and bounds | FIFO_15 – FIFO_17 | Pointer wraparound and pointer/counter limits. |
| UVM flow | FIFO_18 – FIFO_19 | Driving and sampling timing, and end-of-test behavior. |
| Scenario sequences | FIFO_20 – FIFO_27 | Fill, drain, continuous simultaneous access, alternating bursts, reset during operation, data integrity, idle cycles, and rejected operations. |

---

## RTL Issues Identified and Fixed

During verification, five RTL issues were identified and corrected:

| # | Issue | Correction |
|:--:|:---|:---|
| 1 | `wr_ack` and `overflow` were missing from the `rst_n` branch. | Cleared both signals in the reset branch. |
| 2 | The `full && wr_en` condition used bitwise `&` instead of logical `&&`. | Replaced it with logical `&&`. |
| 3 | `underflow` was implemented as a combinational signal. | Registered it in the read block. |
| 4 | `almostfull` compared `count` against `FIFO_DEPTH - 2`. | Corrected the threshold to `FIFO_DEPTH - 1`. |
| 5 | Simultaneous read and write behavior was not handled. | Added the full, empty, and normal-state cases for updating `count`. |

---

## Repository Structure

The repository is organized to keep the complete verification source code together while making the project documentation and supporting reports accessible from the root directory.

```text
Sync-FIFO-Verification-using-UVM/
│
├── Full Environment Code/
│   ├── FIFO.sv
│   ├── FIFO_if.sv
│   ├── FIFO_top.sv
│   ├── shared_pkg.sv
│   ├── FIFO_sva.sv
│   ├── FIFO_config.sv
│   ├── FIFO_seq_item.sv
│   ├── FIFO_rst_seq.sv
│   ├── FIFO_wr_seq.sv
│   ├── FIFO_rd_seq.sv
│   ├── FIFO_wr_rd_seq.sv
│   ├── FIFO_sequencer.sv
│   ├── FIFO_driver.sv
│   ├── FIFO_monitor.sv
│   ├── FIFO_agent.sv
│   ├── FIFO_scoreboard.sv
│   ├── FIFO_coverage.sv
│   ├── FIFO_env.sv
│   ├── FIFO_test.sv
│   ├── src_files.list
│   └── run.do
│
├── FIFO_UVM_documentation_report.pdf
├── UVM.png
├── coverage_report.txt
├── LICENSE
└── README.md
```

### Repository resources

| Resource | Description |
|:---|:---|
| [`Full Environment Code/`](Full%20Environment%20Code/) | Complete SystemVerilog RTL and UVM verification environment, including the testbench components, sequences, assertions, and simulation scripts. |
| [`FIFO_UVM_documentation_report.pdf`](FIFO_UVM_documentation_report.pdf) | Detailed project documentation and verification report. |
| [`UVM.png`](UVM.png) | Visual representation of the UVM testbench architecture. |
| [`coverage_report.txt`](coverage_report.txt) | Simulation coverage report. |
| [`LICENSE`](LICENSE) | Project license (MIT). |

---

## Running the Simulation

### Requirements

- Siemens QuestaSim
- UVM 1.2

### Run

From the `Full Environment Code` directory, execute:

```tcl
vsim -do run.do
```

The `run.do` script compiles the source files listed in `src_files.list`, enables coverage and assertions through `+define+SIM`, runs the simulation, and saves the coverage database.

To generate a coverage report from the saved database, use:

```tcl
vcover report
```

---

## Tools and Methodology

| Category | Technologies |
|:---|:---|
| Hardware description and verification languages | SystemVerilog, SVA |
| Verification methodology | UVM 1.2 |
| Simulation | Siemens QuestaSim |
| Stimulus generation | Constrained-random verification |
| Data checking | Self-checking testbench and golden-model scoreboard |
| Coverage | Functional coverage and DUT code coverage |
| Assertions | Immediate and concurrent SystemVerilog Assertions |

---

## Author

**Haneen Fady Shahin**

[GitHub Profile](https://github.com/Haneen-Shahin)

This project is intended for educational use, learning, and experimentation. You are welcome to explore the implementation and use it as a reference for studying UVM-based FIFO verification.

