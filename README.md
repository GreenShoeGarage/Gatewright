# GATEWRIGHT

Drag and wire basic logic for the Arduino MKR VIDOR 4000 FPGA, then read out
synthesizable Verilog that drops straight into Arduino's `MKRVIDOR4000_top`
template. Place gates, flip-flops and registers on a sheet, connect them to the
MKR header pins, and the tool writes a `user.v` ready for Quartus.

Single file, no build step, no server, no dependencies. Open `gatewright.html`
in a browser and work offline.

## Scope, stated plainly

GATEWRIGHT emits source, not a flashable image. The Cyclone 10 LP on the VIDOR
has no open-source synthesis path, so place and route only happens in Intel
Quartus Prime Lite, which produces the `.ttf` that the SAMD21 loads at boot.
This tool does the part a GUI can actually do well: turn a schematic into clean,
correct Verilog and tell you exactly how to carry it the rest of the way.

The integration point is real. Arduino's `MKRVIDOR4000_top` includes a
`user.v` file through a dedicated include hook. Your sheet becomes that file.
Inside it you have direct access to the board signals:

    iCLK, iRESETn, bMKR_AREF, bMKR_A[6:0], bMKR_D[14:0]

Target device is `10CL016YU256C8G`, family Cyclone 10 LP.

## What it does

The sheet is a live schematic. The panel on the right regenerates four outputs
on every edit:

- `user.v`, the synthesizable Verilog for your design
- `sketch.ino`, a companion sketch that brings the FPGA up and can observe pins
  from the SAMD21 side
- `pin map`, a table of every MKR pin you used, its FPGA ball, its direction,
  and its Verilog net
- `quartus flow`, the step list from this sheet to a running board

A status lamp in the header reads the netlist on every change. It glows green
when the design is clean and turns red with a fix list when it is not.

## Controls

- Arm a part in the left palette, then click the sheet to drop it
- Click an output pad, then an input pad, to wire them. One driver can fan out
  to many inputs. An input takes one driver only.
- Double-click an input or output pin to cycle its MKR pin
- Double-click a register to cycle its width through 2, 4, 8 and 16 bits
- Select a part or wire and press Delete or Backspace to remove it
- Right-click a part or wire to delete it directly
- Ctrl or Cmd with the mouse wheel to zoom, drag empty space to pan, or use the
  zoom controls and the fit button
- Esc cancels a placement or a wire in progress

Work autosaves to the browser. Use Save .json and Open .json to move a design
between machines, Load example for a starting point, and Clear sheet to reset.

## Component reference

Sources and pins

- Input pin. An external signal entering the fabric. Emits an alias,
  `assign w_id = bMKR_X[n];`
- Output pin. The fabric drives this MKR pad. Emits `assign bMKR_X[n] = src;`
- Clock. Maps to `iCLK`, the system clock available in the top module.
- Constant 0 and Constant 1. Emit `1'b0` and `1'b1`.

Logic gates

- AND, OR, XOR, NAND, NOR, XNOR, two inputs each
- NOT and Buffer, one input each

Each gate names its output net and emits a continuous assignment, for example
`assign w_id = a ^ b;` for XOR.

Sequential

- D flip-flop. Ports are D, R, CLK and Q. Leave R unwired for a plain register
  bit. Wire R for an active-high asynchronous clear.
- Register, n bits wide. Exposes per-bit D and Q pads plus a single CLK. Emits a
  vector register and a clocked assignment with bit concatenation.

Nets are single-bit at the wire level. The register exposes its bits as
individual pads, which keeps the netlist provably correct while still giving you
real n-bit state.

## Design checker

The checker runs on every edit and reports through the lamp and the diagnostics
strip. It flags:

- a floating gate input
- an output pin with nothing driving it
- a pin driven by more than one output part
- a pin used as both an input and an output
- a combinational feedback loop, with a note to break it using a flip-flop or
  register
- a register bit left floating, which it ties low with a note
- a flip-flop or register clock left unwired, which it defaults to `iCLK` with a
  note

Notes are advisory and still produce code. Fixes are real errors. The generated
file always compiles to something readable so you can see what the checker saw.

## From sheet to hardware

The `quartus flow` tab carries the full version. In short:

1. Get an Arduino FPGA template project. The VidorFPGA template from
   vidor-libraries, or the wd5gnr tutorial fork, already contains
   `MKRVIDOR4000_top`, the system PLL, and the `user.v` include hook.
2. Drop the generated `user.v` into the project folder, replacing the stub.
3. Open the project in Quartus and confirm the device reads
   `10CL016YU256C8G` and the family reads Cyclone 10 LP.
4. Import Arduino's `vidor_s_pins.qsf` using Assignments, Import Assignments,
   with Pin and Location Assignments only. This populates the Pin Planner with
   the balls shown in the pin map tab.
5. Make sure GENERATE_TTF_FILE is on, then Start Compilation. Quartus writes a
   Tabular Text File.
6. Convert and embed the `.ttf` per the template build script, then upload the
   companion sketch. The SAMD21 loads the fabric at boot and your gates go live
   on the pins.

AREF is an analog reference pad and is left out of the pin choices on purpose.
Use D0 through D14 and A0 through A6 for logic.

## Pin balls

These match Arduino's `vidor_s_pins.qsf`. Do not hand-edit them.

    D0  PIN_G1    D8   PIN_F16    A0  PIN_C2
    D1  PIN_N3    D9   PIN_F15    A1  PIN_C3
    D2  PIN_P3    D10  PIN_C16    A2  PIN_C6
    D3  PIN_R3    D11  PIN_C15    A3  PIN_D1
    D4  PIN_T3    D12  PIN_B16    A4  PIN_D3
    D5  PIN_T2    D13  PIN_C11    A5  PIN_F3
    D6  PIN_G16   D14  PIN_A13    A6  PIN_G2
    D7  PIN_G15

## Known limitations

- Single-bit nets only. There are no bus wires yet, so a register connects bit
  by bit rather than as one fat net.
- No multi-bit gate ports and no bus ripper.
- The D flip-flop has no separate enable. Build one with a multiplexer if you
  need it.
- The companion sketch leaves its FPGA bring-up line as a template-dependent
  comment, because the correct call varies by which VidorBitstream image you
  build against.

## Hosting

Static HTML. Copy `gatewright.html` to any static host, or open it directly from
disk. There is nothing to install and nothing to run server side.

## License

Copyright (C) 2026 Michael B. Parks, Green Shoe Garage.

GNU General Public License v3.0 (GPL-3.0).

This program is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 3 of the License, or (at your option) any later
version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with
this program. If not, it is available from the Free Software Foundation.
