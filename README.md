![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg)

# Four Branch Predictors on one Tiny Tapeout chip

Four branch predictors share one set of pins and see the same branches, so every branch of a program is predicted
four times and the predictors can be compared directly on silicon:

- the **perceptron** from Tiny Tapeout design #704,
- a **bimodal** table,
- **gshare**,
- a **tournament** predictor that picks between bimodal and gshare.

SKY130, Tiny Tapeout `ttsky26d`, 2×2 tiles. **[Open the 3D layout viewer](https://gds-viewer.tinytapeout.com/?pdk=sky130A&model=https%3A%2F%2Fkyrylo-nazarevych.github.io%2Ftt-branch-predictor%2Ftinytapeout.oas)**
(a desktop browser window at least 1000 px wide).

![The chip layout](docs/gds_render.png)

The FPGA version, host tools, the analysis app and the verification data are in
**[quad-branch-predictor](https://github.com/Kyrylo-Nazarevych/quad-branch-predictor)**.

## The chip

| | |
|---|---|
| Cells | 4,701 standard cells, 47,096 µm² (65% of the 2×2 area) |
| Timing | met at 50 MHz (20 ns) in every corner, worst setup slack +0.66 ns |
| Checks | DRC, LVS, precheck and gate-level simulation pass |

| Predictor | Storage | Area (logic) | Answers | Reference trace | `sort_and_search -O1` |
|---|---|---|---|---|---|
| Perceptron (704) | 8 × (bias + 7 weights) × 8 bit in a 64-byte latch RAM, 7-bit history | 16,879 µm² | 11 clocks; 11–45 with training | 81.98% | **84.48%** |
| Bimodal | 64 × 2-bit counters, indexed by PC[7:2] | 7,936 µm² | 2 clocks after the strobe | 66.88% | 84.07% |
| gshare | 64 × 2-bit, PC[7:2] ⊕ 8-bit history folded to 6 bits | 7,948 µm² | 2 clocks | 86.47% | 82.83% |
| Tournament | 64 × 2-bit chooser over bimodal and gshare | 22,948 µm² (all three tables) | 2 clocks | **88.83%** | 83.48% |

The reference trace is the 704 design's own program (1,271 branches); `sort_and_search` is bubble sort plus binary
search (46,922 branches). No predictor wins everywhere, which is the point of having all four on one chip.

## Source files

| File | What |
|---|---|
| [`src/project.v`](src/project.v) | Top level `tt_um_kyrylo_nazarevych_bp`: input registers, the three table predictors, output pins |
| [`src/predictors.v`](src/predictors.v) | Bimodal, gshare and the tournament chooser |
| [`src/perceptron_704.v`](src/perceptron_704.v) | The 704 perceptron core |
| [`src/latch_memory.v`](src/latch_memory.v) | The perceptron's 64-byte latch RAM |
| [`test/test.py`](test/test.py) | cocotb test: replays the reference trace and checks all four predictors (RTL and gate level) |

## Pins

| Pin | Meaning |
|---|---|
| `ui_in[7:0]` | PC[7:0] of the branch |
| `uio_in[0]` | new-branch strobe (rising edge) |
| `uio_in[1]` | the real outcome (1 = taken) |
| `uo_out[0]` / `[1]` | perceptron prediction ready / perceptron prediction |
| `uo_out[2]` | training done: send the next branch |
| `uo_out[3]` | memory reset done (one-clock pulse, about 128 clocks after reset) |
| `uo_out[4]` / `[5]` / `[6]` | bimodal / gshare / tournament prediction |

## How to use it

1. After reset, wait for `uo_out[3]`.
2. For each branch: put the PC on `ui_in` and the outcome on `uio_in[1]`, pulse `uio_in[0]` for one clock and keep
   both stable.
3. Wait for `uo_out[0]`, then read `uo_out[1]` and `uo_out[4..6]`. Every prediction is made before that branch
   trains the predictor.
4. Wait for `uo_out[2]`, then send the next branch.

A microcontroller or an FPGA replays the trace; no other hardware is needed. Test it in simulation with
`cd test && make -B` (needs [cocotb](https://www.cocotb.org/) and Icarus Verilog).

## Authors and credits

Kyrylo Nazarevych, Märten Erik Loor, Ilma Kule and Vitalii Haponenko (Chipathron, October 2026).

- The perceptron predictor is Tiny Tapeout design #704 by Tristan Robitaille
  ([tt_branch_pred](https://github.com/TristanRobitaille/tt_branch_pred), Apache-2.0).
- Its latch RAM is by Michael Bell ([tt06-memory](https://github.com/MichaelBell/tt06-memory), Apache-2.0).
- Built on the [Tiny Tapeout](https://tinytapeout.com) Verilog template.

Apache License 2.0, see [LICENSE](LICENSE) and [NOTICE](NOTICE).
