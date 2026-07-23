# FIFO-calibration-and-control-for-High-Speed-design
## OVERVIEW
In high-speed data transfer design, there is always requirement to convert slow speed clock domain to high-speed clock domain. In some designs, the high-speed data region is high density that it doesn’t allow any logic except for data pipelines. There is also requirement to reduce data latency as much as possible.

The above requirements lead to free running FIFO design (pointer always switches when clock is available), that only converts data from low speed to high speed and removes all normal flags such as empty or full. To use the FIFO properly, the controller must calibrate the pointer, so that the FIFO will never be empty or full.

<img width="1032" height="529" alt="image" src="https://github.com/user-attachments/assets/ef2ecdb5-3522-4488-870f-74c717bb81fa" />

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



## Specifications

<img width="851" height="366" alt="image" src="https://github.com/user-attachments/assets/2080d3b1-ad8a-4a37-babc-89340beafc30" />

Figure. TX Wrapper Architecture and Data/Control Path Integration

<img width="411" height="235" alt="image" src="https://github.com/user-attachments/assets/bda37c14-7da2-455a-a9b2-d86baff39d8f" />

Figure. Long Pipeline


<img width="654" height="324" alt="image" src="https://github.com/user-attachments/assets/7aa4763c-811c-482a-918f-0af482be6ec1" />

Figure. TX FIFO Data Path and Internal Structure

<img width="636" height="390" alt="image" src="https://github.com/user-attachments/assets/9dc22294-0794-4ff5-83d0-86f3a1bb6a42" />

Figure. TX FIFO Memory Mapping (16-bit Word to 4-bit Beats)



<img width="386" height="212" alt="image" src="https://github.com/user-attachments/assets/239857b5-fc6c-4895-9089-487de648db53" />

Figure. TX FIFO Model Delay Block (Propagation Delay)

<img width="526" height="278" alt="image" src="https://github.com/user-attachments/assets/6292ea91-1242-4a89-b7e3-249d1c9ae540" />

Figure. Example of Write/Read Pointer Separation Illustrates pointer distance with margin = 0


<img width="314" height="136" alt="image" src="https://github.com/user-attachments/assets/4f9b1313-4e41-4764-97e8-6bf0166ec034" />
<img width="298" height="138" alt="image" src="https://github.com/user-attachments/assets/aff7956e-8ba1-4c99-a59f-c5d2fc0a308a" />
<img width="607" height="282" alt="image" src="https://github.com/user-attachments/assets/3e2b0235-cc73-4f5d-8663-68c23b01a8b3" />

Figure. TX FIFO calibration block

<img width="469" height="383" alt="image" src="https://github.com/user-attachments/assets/83a03681-8ad9-405a-8633-1d47861db9a9" />

(a) No valid overlap → TxCalOut = 0

<img width="445" height="368" alt="image" src="https://github.com/user-attachments/assets/ee837472-1847-4379-917c-1dde40b671a6" />

(b) Read pointer enters detection window → TxCalOut = 1

Figure. TX FIFO Calibration Timing for Different CalOut Conditions



<img width="507" height="99" alt="image" src="https://github.com/user-attachments/assets/3999f3c1-65b7-451d-8fd5-f8e146b10fb2" />

<img width="363" height="318" alt="image" src="https://github.com/user-attachments/assets/29441c86-3e73-4acc-98fc-cd381ec5632a" />

Figure. Read Clock Pause Logic and Waveform


<img width="350" height="96" alt="image" src="https://github.com/user-attachments/assets/0f1dac5f-3c31-4be0-9bea-5d1cb7f17946" />

Figure. CDC logic


<img width="1014" height="425" alt="image" src="https://github.com/user-attachments/assets/871fc8ff-defc-47e2-ba61-51e65d192a24" />

Figure. Read Pointer Margin Adjustment


<img width="412" height="245" alt="image" src="https://github.com/user-attachments/assets/978a2aa5-7bbf-4f56-9bb0-d306f39be68e" />
<img width="557" height="416" alt="image" src="https://github.com/user-attachments/assets/e2f10f09-5eea-461b-953f-c8b229261a96" />

Figure. TX FIFO TOP Interconnection in the High‑speed Domain


### TX FIFO CTL

<img width="350" height="164" alt="image" src="https://github.com/user-attachments/assets/968c1269-4537-4244-9acb-c58e65120709" />

<img width="596" height="379" alt="image" src="https://github.com/user-attachments/assets/30786dd8-9282-4949-953d-482bfeca850c" />

Figure. TX Calibration Controller Architecture

<img width="296" height="92" alt="image" src="https://github.com/user-attachments/assets/a7a17e3e-1ba5-4824-81d5-dc5376444ae7" />

<img width="996" height="508" alt="image" src="https://github.com/user-attachments/assets/1f539e6d-2946-43c0-b3ee-5441965598f7" />

Figure. CDC Synchronization and Deterministic Delay before CalOut Evaluation (PIPE_DEPTH = 2)

<img width="379" height="150" alt="image" src="https://github.com/user-attachments/assets/3a7a07e0-4dc7-4150-934a-a80dab7eaf63" />

<img width="568" height="379" alt="image" src="https://github.com/user-attachments/assets/15eb7371-a7a6-4ddd-a26c-2176b7653fd7" />


