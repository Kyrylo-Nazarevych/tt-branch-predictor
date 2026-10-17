## How it works

Four branch predictors share one set of pins. For each branch the host puts PC[7:0] on ui_in and the real outcome on uio[1], then raises uio[0]. The bimodal, gshare and tournament predictors (64 two-bit counters each) answer two clocks later on uo[4..6]. The perceptron (the TT IHP 25a #704 design: 8 perceptrons, 7-bit global history, weights in a 64-byte latch RAM) answers when uo[0] goes high, on uo[1]. All four then train on the real outcome, and uo[2] signals that the next branch can be sent.

## How to test

After reset, wait for uo[3] (memory reset done). For each branch of a trace: set ui_in to PC[7:0] and uio[1] to taken, pulse uio[0] for one clock, wait for uo[0], then read uo[1] and uo[4..6], then wait for uo[2]. Compare the predictions with the outcomes to get the accuracy of each predictor.

## External hardware

None. A microcontroller or FPGA replays a branch trace into the pins.
