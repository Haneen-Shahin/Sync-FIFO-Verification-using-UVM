# Synchronous FIFO Verification Using SystemVerilog and UVM

## Overview

This repository houses a comprehensive, industry-standard verification environment engineered for a parameterizable synchronous First-In, First-Out (FIFO) memory subsystem. The architecture leverages **SystemVerilog**, the **Universal Verification Methodology (UVM)**, and concurrent **SystemVerilog Assertions (SVA)** to achieve rigorous pseudo-random and directed verification completeness.

The environment validates data throughput integrity, pointer wrap-around mechanics, flag generation (`full`, `empty`, `almost_full`), and boundary condition protections against overflow and underflow scenarios.

## Design Configuration & Parameters

| Parameter | Configuration / Description |
| :--- | :--- |
| **FIFO Data Width** | 16 bits (parameterizable) |
| **FIFO Depth** | 8 entries (parameterizable) |
| **Architecture Type** | Synchronous FIFO (Single Clock Domain) |
| **Control Flags** | `full`, `empty`, `almost_full`, `almost_empty` with invalid access protection |

## Verification Environment Architecture

The testbench adheres to a strict, modular layered UVM class-based hierarchy, decoupling stimulus generation, virtual interface driving, bus monitoring, reference modeling, and functional coverage extraction.

### Core UVM Components

* **Sequence Item (`fifo_seq_item`)**: Encapsulates all transaction-level abstractions (`wr_en`, `rd_en`, `data_in`, `data_out`, `full`, `empty`) constrained with randomized weights.
* **Sequencer (`fifo_sequencer`)**: Manages transaction arbitration and routes generated sequences downstream to the driver via TLM FIFO channels.
* **Driver (`fifo_driver`)**: Translates high-level UVM transaction packets into pin-level signal toggles across the virtual interface (`fifo_if`).
* **Monitor (`fifo_monitor`)**: Samples pin-level activity, reconstructs bus transactions, and broadcasts them via UVM analysis ports (`uvm_analysis_port`).
* **Scoreboard (`fifo_scoreboard`)**: Implements a high-level transaction-based queue as a golden reference model to perform automated out-of-order/in-order checking and dynamic self-checking assertions.
* **Functional Coverage Collector (`fifo_coverage`)**: Samples cross-coverage bins, transition bins, and toggle metrics for control flags and pointer states.
* **SystemVerilog Assertions (SVA)**: Concurrent and immediate assertions bound directly to the design to verify protocol adherence, illegal state transitions, and zero-latency flag assertions.

## Verification Flow & Execution Pipeline

```text
                    UVM Test Case
                         |
                         v
                    UVM Sequence
                         |
                         v
                    UVM Sequencer
                         |
                         v
                     UVM Driver
                         |
                         v
             +-----------------------+
             |      DUT (RTL)        |
             |   Synchronous FIFO    |
             +-----------------------+
                         |
                         v
                    UVM Monitor
                         |
             +-----------+-----------+
             |                       |
             v                       v
     UVM Scoreboard         Coverage Collector
     (Golden Model)         (Cross/Bins Sched)
             |
             v
   Self-Checking Delta
    (Expected vs Actual)

    Concurrent SVA Checking Active in Background
```

### Simulation Execution Phases

1. **Build & Connect Phase**: Elaborates the UVM component hierarchy (`env`, `agent`, `scoreboard`, `driver`, `monitor`) and establishes TLM port-export connections.
2. **Stimulus Generation**: Sequences generate parameterized bursts, back-to-back operations, and constrained-random delay profiles.
3. **Driving & Sampling**: The driver handles handshaking across the virtual interface while the monitor sniffs bus activity concurrently.
4. **Golden Model Comparison**: The scoreboard consumes packets from both the monitor and reference queue, executing automated mismatch evaluations.
5. **Coverage & Assertion Sampling**: Functional bins track state-space penetration while SVA properties flag protocol violations dynamically at the clock edge.

## Verification Scenarios & Test Plan

* **Asynchronous Reset Verification**: Dynamic assertion and de-assertion of active-low/high resets mid-transaction stream to verify pointer clearing.
* **Directed Burst Operations**: Sequential saturation (write-to-full) followed by draining (read-to-empty).
* **Simultaneous Read/Write (Concurrent Access)**: Validates pointer stability and data throughput when `wr_en` and `rd_en` assert concurrently on full, empty, and intermediate states.
* **Overflow / Underflow Protection**: Injects illegal write commands on full status and read commands on empty status to test error-handling resilience.
* **Constrained-Random Stress Testing**: High-density randomized transaction mixes targeting corner cases and maximizing functional coverage closure.

## Verification Metrics & Results

| Verification Metric | Achieved Result |
| :--- | :--- |
| **Transactions Processed & Checked** | 2,301 |
| **Transaction Mismatches / Data Loss** | 0 |
| **Functional Coverage Bins (FIFO Logic)** | 68 / 68 (100%) |
| **Reset Coverage Bins** | 4 / 4 (100%) |
| **Active SVA Assertions** | 15 Concurrent Properties |

* **Summary**: All 2,301 checked transactions matched the architectural reference model with zero data corruption, achieving **100% functional coverage closure** across defined bins.

## Tech Stack & Verification Tools

* **Languages**: SystemVerilog, UVM 1.2, SVA
* **Methodologies**: Constrained-Random Verification, Scoreboard-Based Self-Checking, Functional Coverage Modeling
* **Simulator**: Siemens QuestaSim

## 👤 Author

**Haneen Fady Shahin**

This project is open for **learning, educational use, and experimentation**. Feel free to study the design, modify it, or use it as a reference for similar academic projects.
