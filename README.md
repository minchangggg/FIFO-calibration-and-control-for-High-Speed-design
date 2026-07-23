# OVERVIEW
This project implements a free-running asynchronous FIFO for high-speed data transfer between different clock domains. Unlike conventional FIFOs, it does not use full or empty flags. Instead, a calibration controller maintains a safe margin between the read and write pointers, enabling continuous, low-latency, and reliable data transfer.

<img width="542" alt="fifo-Architecture drawio" src="https://github.com/user-attachments/assets/58be08ff-d59c-447c-90b2-efd7229aa8c3" />

> Figure High Speed Design

<img width="1000" alt="image" src="https://github.com/user-attachments/assets/112a1181-9bfe-4d18-90e3-226dc8f95281">

> Figure TOP Wrapper of CTL and FIFO

The Top Wrapper connects the CTL, TX FIFO, RX FIFO, and long pipeline blocks. The CTL controls data transfer and calibration for both TX and RX FIFOs, while the pipeline stages help meet timing requirements in high-speed designs. The TX FIFO transfers data from clk to txclk, and the RX FIFO transfers data from rxclk to clk.

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
The TX path transfers data from the system clock domain (clk) to the high-speed transmit clock domain (txclk). To achieve continuous, low-latency data transfer, it uses a free-running asynchronous FIFO without conventional full and empty flags. Instead, a calibration mechanism maintains a safe distance between the write and read pointers, ensuring reliable operation throughout data transmission.

The TX subsystem consists of three main blocks:

- TX Controller (TX CTL) – Generates write, pointer, and calibration control signals.
- TX FIFO – Buffers data and performs clock domain crossing.
- TX FIFO Calibration – Detects the reference point and establishes the required safety margin for normal operation.

The TX calibration process consists of three stages:

- Reference Sweep – The read pointer is gradually shifted until a valid CalOut transition is detected.
- Reference Confirmation – The detected reference point is verified.
- Margin Application – Additional delay is applied to create a safe separation between the read and write pointers.

### TX WRAPPER
The TX Wrapper connects the TX Controller and the TX FIFO while inserting fixed-latency pipeline stages between them. These pipeline stages improve timing closure in high-speed designs, and their deterministic delay is compensated by the calibration logic. After calibration, the TX path can continuously transfer data while maintaining correct pointer alignment and timing across clock domains.

<img width="851" alt="image" src="https://github.com/user-attachments/assets/2080d3b1-ad8a-4a37-babc-89340beafc30" />

> Figure. TX Wrapper Architecture and Data/Control Path Integration

### TX FIFO
The TX FIFO implements a free-running asynchronous FIFO that transfers data from the low-speed write clock domain (WrClk) to the high-speed read clock domain (RdClk).

Unlike a conventional asynchronous FIFO, the write and read pointers run continuously without being stopped by full or empty flags.

The FIFO performs a word‑to‑beat data conversion:

  + A 16-bit word is written into the FIFO in the write clock domain
  + The data is internally stored as four consecutive 4-bit entries
  + The read side outputs one 4-bit beat per read clock cycle.

The FIFO characteristics are as follows:

  + Logical depth of 4 words in the write domain
  + Equivalent to 16 physical 4-bit entries in the read domain.

The FIFO does not provide full or empty status flags. Instead, safe operation is ensured by the calibration mechanism.

The TX calibration mechanism maintains a safe separation between the write and read pointers, ensuring reliable data transfer without conventional flow-control signals.

<img width="654" alt="image" src="https://github.com/user-attachments/assets/7aa4763c-811c-482a-918f-0af482be6ec1" />

> Figure. TX FIFO Data Path and Internal Structure

<img width="526" alt="image" src="https://github.com/user-attachments/assets/6292ea91-1242-4a89-b7e3-249d1c9ae540" />

> Figure. Example of Write/Read Pointer Separation Illustrates pointer distance with margin = 0

### TX FIFO CALIBRATION
The TX FIFO Calibration block determines the relative position between the write and read pointers before normal data transfer begins. Since the FIFO operates without full or empty status flags, calibration is required to establish a safe pointer separation for reliable operation.

<img width="314" alt="image" src="https://github.com/user-attachments/assets/4f9b1313-4e41-4764-97e8-6bf0166ec034" />
<img width="298" alt="image" src="https://github.com/user-attachments/assets/aff7956e-8ba1-4c99-a59f-c5d2fc0a308a" />
<img width="607" alt="image" src="https://github.com/user-attachments/assets/3e2b0235-cc73-4f5d-8663-68c23b01a8b3" />

> Figure. TX FIFO calibration block

The calibration algorithm works by injecting a known data transition into the TX FIFO and monitoring the TxCalOut feedback signal:

- A logic '1' is written into a target FIFO location.
- The data is then changed to logic '0' at the same location.
- If the read pointer enters the detection window, TxCalOut is asserted, indicating the reference position has been found.
- The controller uses this reference point to apply the configured calibration margin before normal operation begins.

This process establishes the required pointer alignment and ensures reliable data transfer without relying on conventional flow-control signals.

<img width="469" alt="image" src="https://github.com/user-attachments/assets/83a03681-8ad9-405a-8633-1d47861db9a9" />

      (a) No valid overlap → TxCalOut = 0

<img width="445" alt="image" src="https://github.com/user-attachments/assets/ee837472-1847-4379-917c-1dde40b671a6" />

      (b) Read pointer enters detection window → TxCalOut = 1

> Figure. TX FIFO Calibration Timing for Different CalOut Conditions

<img width="1200" alt="image" src="https://github.com/user-attachments/assets/871fc8ff-defc-47e2-ba61-51e65d192a24" />

> Figure. Read Pointer Margin Adjustment

### TX FIFO CONNECTION
The TX FIFO TOP connects multiple TX FIFO instances with a shared pointer scheme and a dedicated TX FIFO Calibration block. All FIFOs use the same write and read pointers, ensuring consistent data ordering, synchronized pointer advancement, and uniform calibration across the entire TX datapath.

<img width="412" alt="image" src="https://github.com/user-attachments/assets/978a2aa5-7bbf-4f56-9bb0-d306f39be68e" />
<img width="557" alt="image" src="https://github.com/user-attachments/assets/e2f10f09-5eea-461b-953f-c8b229261a96" />

> Figure. TX FIFO TOP Interconnection in the High‑speed Domain

### TX FIFO CTL
The TX Controller (TX CTL) manages the TX FIFO calibration process and maintains a safe separation between the write and read pointers. Since the FIFO operates without full and empty status flags, the controller ensures reliable data transfer by coordinating pointer movement and calibration.

The TX Controller is divided into three functional blocks:

- TX Write Generator – Generates write data and advances the write pointer.
- TX Calibration Sequencer – Controls the calibration flow and determines the reference pointer position.
- TX Calibration Helper – Processes synchronized calibration feedback and generates the required read-clock pause commands.

<img width="350" alt="image" src="https://github.com/user-attachments/assets/968c1269-4537-4244-9acb-c58e65120709" />
<img width="596" alt="image" src="https://github.com/user-attachments/assets/30786dd8-9282-4949-953d-482bfeca850c" />

> Figure. TX Calibration Controller Architecture

All communication between the TX Controller and the TX FIFO passes through fixed-latency pipeline stages. During calibration, the controller compensates for both pipeline latency and CDC synchronization latency before evaluating TxCalOut, ensuring that the feedback is stable and correctly aligned with the corresponding write operation.

<img width="296" alt="image" src="https://github.com/user-attachments/assets/a7a17e3e-1ba5-4824-81d5-dc5376444ae7" />

<img width="996" alt="image" src="https://github.com/user-attachments/assets/1f539e6d-2946-43c0-b3ee-5441965598f7" />

> Figure. CDC Synchronization and Deterministic Delay before CalOut Evaluation (PIPE_DEPTH = 2)

The waveform illustrates the timing relationship between the write operation, TxCalOut feedback, pipeline latency, and CDC synchronization latency. The controller delays the calibration evaluation until all deterministic latencies have elapsed, ensuring that the synchronized feedback corresponds to the correct write transaction.

#### TX WRITE GENERATION
The TX Write Generator produces deterministic write data and advances the write pointer during calibration. Its outputs are synchronized with the calibration sequence through start_calib and wrptr_tick, ensuring repeatable write operations for reference detection.

<img width="330" alt="image" src="https://github.com/user-attachments/assets/3a7a07e0-4dc7-4150-934a-a80dab7eaf63" />

<img width="500" alt="image" src="https://github.com/user-attachments/assets/c5c76633-fbff-48c9-937e-3c7ecb289dfb" />

> Figure. TX write generation block diagram

#### TX CALIBRATION SEQUENCER
The TX Calibration Sequencer controls the overall calibration flow by coordinating the calibration stages and monitoring the TxCalOut feedback. Based on the calibration result, it determines whether to continue, retry, or complete the calibration process.

<img width="350" alt="image" src="https://github.com/user-attachments/assets/db721d15-092d-422a-9799-49ee55f89fb2" />
<img width="700" alt="image" src="https://github.com/user-attachments/assets/190add2f-7180-417a-9f77-29207ccb3375" />

> Figure. TX calibration sequencer block diagram

<img width="1207" alt="image" src="https://github.com/user-attachments/assets/73e42bd3-a172-410a-beac-8ac4fbf5c154" />

      (a) The calibration start is triggered by an external enable request (txcalib_en)

<img width="1254" alt="image" src="https://github.com/user-attachments/assets/d2550b9a-a4d1-4756-8b1e-3d96df2fd35e" />

      (b) Calibration retry sequence when no valid reference is found after a calibration attempt

> Figure. Timing Diagram of TX Calibration Sequencer


#### TX CALIBRATION HELPER
The TX Calibration Helper processes the synchronized TxCalOut feedback and generates the required read-clock pause commands to support pointer alignment during calibration.

<img width="430" alt="image" src="https://github.com/user-attachments/assets/e1cf6e73-8908-41ce-94c2-1d6449b7ba33" />

<img width="600" alt="image" src="https://github.com/user-attachments/assets/6e3c8bec-2077-44c7-9f8e-f5771278fb50" />

> Figure. TX Calibration Helper Block Diagram

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

<img width="900" alt="rx_dv_may2-Page-5" src="https://github.com/user-attachments/assets/61533fee-f4e6-426a-ad11-dc07abe6e4b0" />

<img width="800" alt="rx_dv_may2-Page-5 (1)" src="https://github.com/user-attachments/assets/ad729a13-ea86-4c2e-b954-bcfbee56e177" />

> Figure. RX Testbench File Structure

## Verification testbench architecture
The verification testbench architecture is designed as a modular and reusable environment, which is shared for RX calibration verification.

At the top level (TB_TOP), the testbench integrates the Device Under Test (DUT), interface, and checker logic (SVA) to validate functionality and protocol behavior.

<img width="800" alt="rx_dv_may2-Page-5 (2)" src="https://github.com/user-attachments/assets/9723667d-5811-4baf-9186-612da3c8418b" />

> Figure. Verification testbench architecture

## RX Verification Results 
All regression tests passed successfully

No SVA violations were detected during verification

<img width="800" alt="image" src="https://github.com/user-attachments/assets/d540c645-d96e-4f03-9a6f-6423a982347b" />

> Figure. RX regression test results summary

Achieved 100% functional coverage across all variables and cross coverage

No uncovered bins observed in any coverage group

→ Confirms completeness of verification for RX calibration behavior

<img width="400" alt="image" src="https://github.com/user-attachments/assets/d4417231-fde0-491a-a76f-5af73e8c25b8" />

<img width="500" alt="image" src="https://github.com/user-attachments/assets/a754a8b4-f9b4-49e1-a1b0-0863d82c1391" />

> Figure. RX calibration functional coverage summary









