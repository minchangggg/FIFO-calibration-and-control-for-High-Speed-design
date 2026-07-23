# OVERVIEW
In high-speed data transfer design, there is always requirement to convert slow speed clock domain to high-speed clock domain. In some designs, the high-speed data region is high density that it doesn’t allow any logic except for data pipelines. There is also requirement to reduce data latency as much as possible.

The above requirements lead to free running FIFO design (pointer always switches when clock is available), that only converts data from low speed to high speed and removes all normal flags such as empty or full. To use the FIFO properly, the controller must calibrate the pointer, so that the FIFO will never be empty or full.

<img width="542" alt="fifo-Architecture drawio" src="https://github.com/user-attachments/assets/58be08ff-d59c-447c-90b2-efd7229aa8c3" />

> Figure High Speed Design

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/112a1181-9bfe-4d18-90e3-226dc8f95281">

> Figure TOP Wrapper of CTL and FIFO

The TOP Wrapper provides the top-level integration of the Control Unit (CTL) with the Transmit (TX) and Receive (RX) FIFO blocks and Long Pipeline blocks. It serves as a structural wrapper responsible for signal routing, timing isolation, and clock-domain interfacing, without implementing protocol or algorithmic logic.
The CTL block generates control, pointer, and calibration signals for both TX and RX paths, and receives corresponding calibration and status outputs from the FIFO blocks. All control and status signals between CTL and FIFO TX/RX are routed through dedicated long pipeline stages to address long-path timing constraints and physical implementation requirements.
The CTL block generates control, pointer, and calibration signals for both TX and RX paths, and receives corresponding calibration and status outputs from the FIFO blocks. All control and status signals between CTL and FIFO TX/RX are routed through dedicated long pipeline stages to address long-path timing constraints and physical implementation requirements.
The TX FIFO operates using the system clock ( clk ) and transmit clock ( txclk ), buffering transmit data and supporting TX calibration. The RX FIFO operates using the system clock ( clk ) and receive clock ( rxclk ), capturing and buffering receive data and supporting RX calibration. Pipeline stages are applied on both forward and return paths to ensure proper signal alignment and timing closure across clock domains.

### Signals for CTL Interface Description

> CTL Control Ports

| Pin Name | Direction | Clock | Frequency | Description |
|---|---|---|---|---|
| `clk` | Input | — | 2 GHz | Control clock |
| `txclk` | Input | — | 8 GHz | High-speed clock for TX |
| `rxclk` | Input | — | 8 GHz | High-speed clock for RX |
| `rst_n` | Input | Async | — | Asynchronous reset, active low |
| `init` | Input | `clk` | — | Synchronous initialization, active high |

> Calibration Control Ports

| Pin Name | Direction | Clock | Frequency | Description |
|---|---|---|---|---|
| `txcalib_en` | Input | `clk` | — | Enables TX calibration |
| `txcalib_done` | Output | `clk` | — | Indicates completion of TX calibration |
| `txcalib_margin[3:0]` | Input | `clk` | — | Target margin for TX calibration |
| `rxcalib_en` | Input | `clk` | — | Enables RX calibration |
| `rxcalib_done` | Output | `clk` | — | Indicates completion of RX calibration |
| `rxcalib_margin[3:0]` | Input | `clk` | — | Target margin for RX calibration |
| `RdPtr_latency[2:0]` | Input | `clk` | — | Pipeline depth of the RX read-pointer path |
| `RxCalOut_latency[2:0]` | Input | `clk` | — | Pipeline depth of the `RxCalOut` feedback path |
| `CalCtl_latency[2:0]` | Input | `clk` | — | Pipeline depth of the `rx_cal_ctl` control path |

> FIFO Control Ports

| Pin Name | Direction | Clock | Frequency | Description |
|---|---|---|---|---|
| `tx_wrdata[15:0]` | Output | `clk` | — | TX marker data written during calibration |
| `tx_wrptr[1:0]` | Output | `clk` | — | TX write pointer |
| `tx_cal_ctl[n-1:0]` | Output | `clk` | — | TX calibration control signals |
| `tx_calout` | Input | `txclk` | — | TX calibration feedback signal |
| `rx_rdptr[1:0]` | Output | `clk` | — | RX read pointer |
| `rx_cal_ctl[n-1:0]` | Output | `clk` | — | RX calibration control signals |
| `rx_calout[3:0]` | Input | `clk` | — | RX calibration feedback value |

# TX DESIGN
## TX Specifications
| Features

- The TX path is responsible for transferring data from the low-speed control clock domain (clk) to the high-speed transmit clock domain (txclk).
- To minimize latency and reduce logic complexity in the high-speed region, a free-running FIFO architecture is adopted, in which conventional full and empty flags are removed.
- Since both write and read pointers advance continuously with their respective clocks, a dedicated calibration mechanism is required to establish and maintain safe separation between pointers.
- The TX subsystem therefore consists of the TX FIFO, a FIFO calibration block, and a TX control (TX CTL) unit, all interconnected through deterministic long pipelines to meet timing constraints.

| TX Calibration Flow Summary

- TX calibration proceeds in three phases: reference sweep, reference confirmation, and margin application.
  + During the reference sweep, the read pointer is gradually shifted using discrete read-clock pauses until a valid CalOut transition is detected.
  + Once the reference point is confirmed, additional pauses corresponding to the programmed calibration margin are applied to establish a safe pointer separation for normal operation.

### TX WRAPPER
The TX Wrapper provides the top-level structural integration between the TX Controller (TX CTL) and the TX FIFO blocks. It does not implement protocol or algorithmic logic, but instead focuses on signal routing, timing isolation, and clock-domain interfacing.

A fixed-latency control pipeline is inserted between TX CTL and the TX FIFO Top to close timing on long control paths. This pipeline introduces a deterministic delay, which is explicitly accounted for by the TX calibration logic to ensure correct alignment of write, pause, and calibration feedback signals.

Before calibration, the relative position between write and read pointers is undefined, and safe data transfer is not guaranteed.

<img width="851" alt="image" src="https://github.com/user-attachments/assets/2080d3b1-ad8a-4a37-babc-89340beafc30" />

> Figure. TX Wrapper Architecture and Data/Control Path Integration

<img width="411" alt="image" src="https://github.com/user-attachments/assets/bda37c14-7da2-455a-a9b2-d86baff39d8f" />

> Figure. Long Pipeline


### TX FIFO
| Overview

- The TX FIFO implements a free‑running asynchronous FIFO to transfer data from the low‑speed control clock domain ( WrClk ) to the high‑speed transmit clock domain ( RdClk ).
- The FIFO performs a word‑to‑beat data conversion:
  + A 16‑bit write word is written into the FIFO in the control domain
  + The data is internally stored as four consecutive 4‑bit memory entries
  + The read side outputs 4‑bit data beats at high speed
- The FIFO characteristics are as follows:
  + Logical depth of 4 words in the write domain
  + Equivalent to 16 physical entries in the read domain

- The FIFO does not provide explicit full/empty status and therefore relies entirely on calibration to guarantee safe operation.
- A safe pointer separation is maintained through the TX calibration mechanism to ensure reliable data transfer in the absence of flow-control signals.

<img width="654" alt="image" src="https://github.com/user-attachments/assets/7aa4763c-811c-482a-918f-0af482be6ec1" />

> Figure. TX FIFO Data Path and Internal Structure

<img width="636" alt="image" src="https://github.com/user-attachments/assets/9dc22294-0794-4ff5-83d0-86f3a1bb6a42" />

> Figure. TX FIFO Memory Mapping (16-bit Word to 4-bit Beats)

<img width="386" alt="image" src="https://github.com/user-attachments/assets/239857b5-fc6c-4895-9089-487de648db53" />

> Figure. TX FIFO Model Delay Block (Propagation Delay)

<img width="526" alt="image" src="https://github.com/user-attachments/assets/6292ea91-1242-4a89-b7e3-249d1c9ae540" />

> Figure. Example of Write/Read Pointer Separation Illustrates pointer distance with margin = 0

<img width="314" alt="image" src="https://github.com/user-attachments/assets/4f9b1313-4e41-4764-97e8-6bf0166ec034" />
<img width="298" alt="image" src="https://github.com/user-attachments/assets/aff7956e-8ba1-4c99-a59f-c5d2fc0a308a" />
<img width="607" alt="image" src="https://github.com/user-attachments/assets/3e2b0235-cc73-4f5d-8663-68c23b01a8b3" />

> Figure. TX FIFO calibration block

<img width="469" alt="image" src="https://github.com/user-attachments/assets/83a03681-8ad9-405a-8633-1d47861db9a9" />

      (a) No valid overlap → TxCalOut = 0

<img width="445" alt="image" src="https://github.com/user-attachments/assets/ee837472-1847-4379-917c-1dde40b671a6" />

      (b) Read pointer enters detection window → TxCalOut = 1

> Figure. TX FIFO Calibration Timing for Different CalOut Conditions

<img width="507" alt="image" src="https://github.com/user-attachments/assets/3999f3c1-65b7-451d-8fd5-f8e146b10fb2" />
<img width="363" alt="image" src="https://github.com/user-attachments/assets/29441c86-3e73-4acc-98fc-cd381ec5632a" />

> Figure. Read Clock Pause Logic and Waveform

<img width="350" alt="image" src="https://github.com/user-attachments/assets/0f1dac5f-3c31-4be0-9bea-5d1cb7f17946" />

> Figure. CDC logic

<img width="1014" alt="image" src="https://github.com/user-attachments/assets/871fc8ff-defc-47e2-ba61-51e65d192a24" />

> Figure. Read Pointer Margin Adjustment

### TX FIFO CONNECTION
In the high-speed transmit clock domain, multiple FIFO memory instances are interconnected to ensure that all memory banks share a consistent read and write pointer value.

This interconnection guarantees correct data ordering and coherent pointer advancement across the entire TX FIFO structure.

Furthermore, the unified pointer scheme enables consistent data alignment and accurate calibration across the entire TX data path.

<img width="412" alt="image" src="https://github.com/user-attachments/assets/978a2aa5-7bbf-4f56-9bb0-d306f39be68e" />
<img width="557" alt="image" src="https://github.com/user-attachments/assets/e2f10f09-5eea-461b-953f-c8b229261a96" />

> Figure. TX FIFO TOP Interconnection in the High‑speed Domain

### TX FIFO CTL
The TX Controller (TX CTL) is responsible for coordinating TX FIFO calibration and maintaining a safe separation between the write pointer and the read pointer in a free-running FIFO architecture.

In the absence of FIFO status flags, TX CTL effectively serves as a flow-control mechanism by regulating pointer separation.

The controller operates entirely in the control clock domain (clk) and interfaces with the TX FIFO and TX FIFO calibration blocks through fixed-latency pipelines.

Since the FIFO does not provide explicit full or empty indicators, TX CTL establishes a valid operating point using deterministic write behavior, synchronized calibration feedback, and controlled read-clock pauses.

To achieve modularity, timing robustness, and scalability, the TX CTL is partitioned into three functionally independent blocks:
- TX Write Generator
- TX Calibration Sequencer
- TX Calibration Helper

Each block has a clearly defined responsibility and communicates with the others using well-scoped control and status signals.

<img width="350" alt="image" src="https://github.com/user-attachments/assets/968c1269-4537-4244-9acb-c58e65120709" />

<img width="596" alt="image" src="https://github.com/user-attachments/assets/30786dd8-9282-4949-953d-482bfeca850c" />

> Figure. TX Calibration Controller Architecture

The internal architecture of the TX Controller follows a strict separation of concerns:

- Data pattern generation and write-pointer activity are isolated in the TX Write Generator.
- Calibration session control, timing alignment, and retry logic are handled by the TX Calibration Sequencer.
- Interpretation of synchronized calibration feedback and generation of step-based pause control are implemented in the TX Calibration Helper.

This separation allows each block to be independently validated, simplifies timing closure across long control paths, and enables future extension of the calibration algorithm with minimal coupling.

All control and status signals exchanged between the TX Controller and TX FIFO pass through deterministic long pipelines. As a result, the TX Controller must explicitly account for various sources of latency, including control pipeline depth (tx_pipe_depth), clock-domain crossing (CDC) latency, and synchronizer delays associated with calibration feedback. Consequently, calibration decisions are not made based on immediate signal values, but only after the corresponding pipeline and synchronization delays have been fully compensated. This ensures that the TX Controller evaluates calibration feedback only when it is stable and correctly aligned with the associated write operation.

The overall timing relationship between write operations, calibration feedback, and synchronized evaluation is illustrated in the waveform below.

<img width="296" alt="image" src="https://github.com/user-attachments/assets/a7a17e3e-1ba5-4824-81d5-dc5376444ae7" />

<img width="996" alt="image" src="https://github.com/user-attachments/assets/1f539e6d-2946-43c0-b3ee-5441965598f7" />

> Figure. CDC Synchronization and Deterministic Delay before CalOut Evaluation (PIPE_DEPTH = 2)

As illustrated in Figure, the calibration feedback signal (CalOut) is generated in the TX FIFO Calibration block operating in the high-speed clock domain and is transferred to the TX controller clock domain through a CDC synchronizer. Due to the use of a two-stage flip-flop synchronizer, the synchronized signal (CalOut_syn) becomes visible in the control domain only after a fixed synchronization latency. 

To ensure accurate evaluation of the calibration feedback, the TX Controller intentionally inserts a fixed delay between the calibration write event and the CalOut checking point. This delay accounts for both pipeline latency and CDC synchronization latency, ensuring that CalOut is sampled only when it is stable and properly aligned with the corresponding write operation.

Therefore, starting from the assertion of start_calib in the controller, CalOut evaluation is enabled after (PIPE_DEPTH + 4) clock cycles. For example, when PIPE_DEPTH = 2, as shown in Figure, calibration is initiated at T1, and CalOut is correctly evaluated at T7. 

This timing-aware design ensures that calibration decisions are made only when feedback signals are stable and correctly aligned with their corresponding write operations.

#### TX WRITE GENERATION
<img width="350" alt="image" src="https://github.com/user-attachments/assets/3a7a07e0-4dc7-4150-934a-a80dab7eaf63" />

<img width="568" alt="image" src="https://github.com/user-attachments/assets/15eb7371-a7a6-4ddd-a26c-2176b7653fd7" />

<img width="500" alt="image" src="https://github.com/user-attachments/assets/c5c76633-fbff-48c9-937e-3c7ecb289dfb" />


#### TX CALIBRATION SEQUENCER
<img width="400" alt="image" src="https://github.com/user-attachments/assets/db721d15-092d-422a-9799-49ee55f89fb2" />
<img width="750" alt="image" src="https://github.com/user-attachments/assets/190add2f-7180-417a-9f77-29207ccb3375" />

> Figure. TX calibration sequencer block diagram

<img width="500" alt="image" src="https://github.com/user-attachments/assets/19c259db-bdec-45db-8717-294a1c2babd5" />

      (a) CalibSessionCtrl Sub block diagram

<img width="550" alt="image" src="https://github.com/user-attachments/assets/40ed548b-51b5-4547-9b46-515ccdfcf5a4" />

      (b) CalibStartCtrl Sub block diagram

<img width="480" alt="image" src="https://github.com/user-attachments/assets/6258be76-becb-4c02-8c15-7275e0092e84" />

      (c) CalOutCheck Sub block diagram

<img width="680" alt="image" src="https://github.com/user-attachments/assets/ce4e6c92-9c70-4700-beba-2018b5da1bbb" />

      (d) CalibEndCtrl Sub block diagram

<img width="380" alt="image" src="https://github.com/user-attachments/assets/854bb4a6-ee78-4e75-8c5d-ca4cb44e9c9c" />

      (e) CalOutInitCtrl Sub block diagram


<img width="1207" alt="image" src="https://github.com/user-attachments/assets/73e42bd3-a172-410a-beac-8ac4fbf5c154" />

      (a) The calibration start is triggered by an external enable request (txcalib_en)

<img width="1254" alt="image" src="https://github.com/user-attachments/assets/d2550b9a-a4d1-4756-8b1e-3d96df2fd35e" />

      (b) Calibration retry sequence when no valid reference is found after a calibration attempt

> Figure. Timing Diagram of TX Calibration Sequencer


#### TX CALIBRATION HELPER

<img width="430" alt="image" src="https://github.com/user-attachments/assets/e1cf6e73-8908-41ce-94c2-1d6449b7ba33" />

<img width="600" alt="image" src="https://github.com/user-attachments/assets/6e3c8bec-2077-44c7-9f8e-f5771278fb50" />

> Figure. TX Calibration Helper Block Diagram


<img width="750" alt="image" src="https://github.com/user-attachments/assets/ce8bd4ec-9bf5-4675-8fe2-26327f75c36b" />

      (a) RefFoundDetector Sub block diagram

<img width="500" alt="image" src="https://github.com/user-attachments/assets/56da0df6-b8b8-4800-a927-92d16a381297" />

      (b) PauseLengthSelect Sub block diagram

<img width="840" alt="image" src="https://github.com/user-attachments/assets/e09af396-efb9-43c6-8c67-d6f5e922aa58" />

      (c) PauseStepCtrl Sub block diagram


<img width="1231" alt="image" src="https://github.com/user-attachments/assets/27da3769-6f9f-4f9d-926d-aa97e328a01e" />

      (a) Qualified CalOut Rising Edge Detection (Reference Found)

<img width="1057" alt="image" src="https://github.com/user-attachments/assets/673b25d5-2b1b-4a4d-90bd-db9894abe28c" />

      (b) Ignored CalOut Transition During First Calibration

> Figure. Timing Diagram of TX Calibration Helper

## TX Simulation Results  
<img width="1462" alt="image" src="https://github.com/user-attachments/assets/c27246a9-d6a7-472d-a49f-b77dd03ed886" />

      a) Initial TX Calibration Phase
      
<img width="1442" alt="image" src="https://github.com/user-attachments/assets/b7c62371-668a-425b-8825-2961c9d421ae" />

      b) Detect the reference point via CalOut and adjust it according to the input margin

<img width="1461" alt="image" src="https://github.com/user-attachments/assets/69e8b956-a6a5-4009-bf59-f7b5425b7e8a" />

      c) Stable TX Operation After Calibration

> Figure. TX Block Simulation Results

# RX VERIFICATION

## File structure

<img width="5000" alt="rx_dv_may2-Page-5" src="https://github.com/user-attachments/assets/61533fee-f4e6-426a-ad11-dc07abe6e4b0" />

<img width="2500" alt="rx_dv_may2-Page-5 (1)" src="https://github.com/user-attachments/assets/ad729a13-ea86-4c2e-b954-bcfbee56e177" />

> Figure. RX Testbench File Structure

## Verification testbench architecture
The verification testbench architecture is designed as a modular and reusable environment, which is shared for RX calibration verification.

At the top level (TB_TOP), the testbench integrates the Device Under Test (DUT), interface, and checker logic (SVA) to validate functionality and protocol behavior.

<img width="4500" alt="rx_dv_may2-Page-5 (2)" src="https://github.com/user-attachments/assets/9723667d-5811-4baf-9186-612da3c8418b" />

> Figure. Verification testbench architecture

#### RX Clock Generator (VIP)
- A dedicated rxclk generator (VIP) provides clock stimulus:
  + Generates configurable clock (frequency, phase, start timing)
  + Operates independently from stimulus generation
  + Plays a critical role in calibration, since clock phase affects: CalOut detection, pointer alignment, final calib margin

#### RX Scoreboard
<img width="600" alt="image" src="https://github.com/user-attachments/assets/12a58dd8-c89c-4989-9fb5-bcfd6a3b617f" />

```
At T1: rdptr = 3, wrptr = 2 => raw_margin = (2 - 4*4) = -14 => real_margin = -14 + 16 = 2
At T2: rdptr = 0, wrptr = 6 => raw_margin = (6 - 4*1) = 2 => real_margin = 2
```

<img width="7000" alt="rx_dv_may2-Page-5 (5)" src="https://github.com/user-attachments/assets/9c555e1a-fa34-4841-9f4c-1349ac49d2f3" />

<img width="7000" alt="rx_dv_may2-Page-5 (4)" src="https://github.com/user-attachments/assets/92d7cbb3-4340-4e6f-a736-4264fda5c486" />

Figure. Scoreboard Margin Calculation and Validation (margin = 2)

#### RX Coverage
Objective
- Verify calibration behavior across CalOut, margin, and latency
- Ensure correctness under all operating conditions

Sampling

- When: When rxcalib_done rising edge
- Why: stable, final calibration result (latched CalOut)

Coverage Closure

- Directed tests → hit basic bins
- Random tests → expand space
- Closure test → fill gaps

=> Target: 100% cross coverage

<img width="659" alt="image" src="https://github.com/user-attachments/assets/10d0b281-9b11-43af-aa5c-89d31be432ac" />

<img width="656" alt="image" src="https://github.com/user-attachments/assets/fa93f25c-92fc-414e-8cf2-eb1f4b249def" />

## RX Verification Results 
All regression tests passed successfully

No SVA violations were detected during verification

<img width="1212" alt="image" src="https://github.com/user-attachments/assets/d540c645-d96e-4f03-9a6f-6423a982347b" />

> Figure. RX regression test results summary

Achieved 100% functional coverage across all variables and cross coverage

No uncovered bins observed in any coverage group

→ Confirms completeness of verification for RX calibration behavior

<img width="525" alt="image" src="https://github.com/user-attachments/assets/d4417231-fde0-491a-a76f-5af73e8c25b8" />

<img width="940" alt="image" src="https://github.com/user-attachments/assets/a754a8b4-f9b4-49e1-a1b0-0863d82c1391" />

> Figure. RX calibration functional coverage summary









