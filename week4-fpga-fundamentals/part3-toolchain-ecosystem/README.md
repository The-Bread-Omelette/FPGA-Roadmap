# Part 3 — The Toolchain Ecosystem

> The tools you use to develop FPGA hardware are as important as the hardware itself. Understanding what each tool does — and why — prevents hours of confusion.

---

## The Full FPGA Development Stack

```
Your Verilog/VHDL Code
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│                  SYNTHESIS                              │
│  Converts HDL to a gate-level netlist                   │
│  Tool: Vivado Synthesis / Yosys                         │
└─────────────────────────────────────────────────────────┘
         │
         ▼ Netlist (.edf, .ngc, .dcp)
┌─────────────────────────────────────────────────────────┐
│              IMPLEMENTATION                             │
│  ┌──────────────────────────────────────────────────┐   │
│  │  MAPPING — assign gates to FPGA primitives       │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │  PLACING — decide physical location of each cell │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │  ROUTING — connect cells with wires              │   │
│  └──────────────────────────────────────────────────┘   │
│  Tool: Vivado Implementation                            │
└─────────────────────────────────────────────────────────┘
         │
         ▼ Placed & routed design (.dcp)
┌─────────────────────────────────────────────────────────┐
│           TIMING ANALYSIS                               │
│  Verify all paths meet timing constraints               │
│  Tool: Vivado Static Timing Analysis                    │
└─────────────────────────────────────────────────────────┘
         │
         ▼ (if timing passes)
┌─────────────────────────────────────────────────────────┐
│         BITSTREAM GENERATION                            │
│  Produce the binary file that programs the FPGA         │
│  Tool: Vivado write_bitstream                           │
└─────────────────────────────────────────────────────────┘
         │
         ▼ .bit file
┌─────────────────────────────────────────────────────────┐
│           PROGRAMMING                                   │
│  Load bitstream onto FPGA via JTAG                      │
│  Tool: Vivado Hardware Manager / OpenOCD                │
└─────────────────────────────────────────────────────────┘
         │
         ▼
    Running Hardware
```

---

## Xilinx / AMD Vivado

**Vivado** is AMD/Xilinx's primary development environment for 7-series and newer devices.

**What Vivado does:**
- HDL synthesis (converts Verilog/VHDL to netlist)
- Implementation (map, place, route)
- Static timing analysis
- Bitstream generation
- IP catalog (reusable hardware modules)
- Block Design editor (graphical IP integration)
- Simulation (Vivado Simulator, or integration with ModelSim/Questa)
- Hardware Manager (programming the device via JTAG)
- Logic Analyzer / ILA (in-system debugging)

**Versions:**
- Vivado ML Edition (current, replaces Design Edition)
- Supports: 7-series, UltraScale, UltraScale+, Versal
- Does NOT support older ISE-era devices (Spartan-3/6, Virtex-4/5/6)

**License:**
- Vivado ML Standard Edition: free, supports most 7-series devices
- Vivado ML Enterprise: paid, required for some UltraScale+ devices

---

## Xilinx / AMD Vitis

**Vitis** is the software development environment for Zynq and other processor-based devices.

Where Vivado handles the FPGA (PL — Programmable Logic) side, Vitis handles the processor (PS — Processing System) side.

**What Vitis does:**
- C/C++ software development for ARM/MicroBlaze processors
- Hardware/software co-design (integrates with Vivado-generated hardware)
- Board Support Package (BSP) generation
- Bare-metal and Linux application development
- AI/ML inference deployment (with Vitis AI)
- IP development and validation

**Vitis Unified IDE** (2022.2+): The newer interface that unifies Vitis Classic and Vitis HLS into one environment. This course covers the Unified IDE.

---

## iCEstudio & Open-Source Tools

For those who want an open-source alternative (or are using Lattice iCE40/ECP5 devices):

### Yosys

Open-source synthesis tool. Supports Verilog (and some SystemVerilog). Used as the synthesis engine in the open-source FPGA toolchain.

```bash
# Install
sudo apt install yosys

# Run synthesis
yosys -p "synth_ice40 -top top -json out.json" design.v
```

### nextpnr

Open-source place and route tool. Supports iCE40, ECP5, and other Lattice devices.

### IceStorm / Project Trellis

Open-source bitstream tools for iCE40 and ECP5 respectively.

### iCEstudio

Visual IDE built on the open-source stack. Good for learning FPGA basics without the complexity of Vivado.

**Why the open-source stack matters:** The Xilinx/Intel tools are free to download but closed-source. For academic research, security-critical designs, or just understanding every step of the flow, open-source tools offer full transparency.

---

## Constraint Files — XDC

The **XDC (Xilinx Design Constraints)** file is one of the most important files in your project. It tells Vivado:

1. **Pin constraints** — which physical FPGA pin corresponds to each signal in your design
2. **Timing constraints** — what clock frequency you are targeting
3. **I/O standards** — what voltage levels the I/O signals use

Without the XDC file, Vivado does not know where to connect your signals on the chip.

```tcl
# Example XDC for a Nexys A7 board

# Clock: 100 MHz on pin E3
create_clock -name sys_clk -period 10.00 -waveform {0 5} [get_ports clk]
set_property PACKAGE_PIN E3     [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports clk]

# LEDs
set_property PACKAGE_PIN H17    [get_ports {led[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {led[0]}]
set_property PACKAGE_PIN K15    [get_ports {led[1]}]
set_property IOSTANDARD LVCMOS33 [get_ports {led[1]}]

# Switches
set_property PACKAGE_PIN J15    [get_ports {sw[0]}]
set_property IOSTANDARD LVCMOS33 [get_ports {sw[0]}]
```

**Where to get pin assignments:**

Every FPGA board comes with a master XDC file listing all pins. Find yours:
- Nexys A7: https://github.com/Digilent/digilent-xdc
- Basys 3: https://github.com/Digilent/digilent-xdc
- Arty A7: same repo

Download the board's master XDC, uncomment the pins you need, and modify the port names to match your Verilog module.

---

## Tcl — The Scripting Language Under Vivado

Vivado uses **Tcl (Tool Command Language)** for automation. Every button click in Vivado's GUI generates equivalent Tcl commands, which you can see in the Tcl console.

This matters because:
- You can script an entire build flow (`vivado -mode batch -source build.tcl`)
- You can parameterize builds (build for 5 different boards with one script)
- CI/CD pipelines for hardware use Tcl batch mode
- All Vivado documentation shows both GUI steps and Tcl equivalents

Basic Tcl commands you will use:

```tcl
# Open project
open_project ./my_project/my_project.xpr

# Run synthesis
launch_runs synth_1 -jobs 8
wait_on_run synth_1

# Run implementation
launch_runs impl_1 -to_step write_bitstream -jobs 8
wait_on_run impl_1

# Open hardware manager
open_hw_manager
connect_hw_server
open_hw_target
program_hw_devices [get_hw_devices xc7a100t_0]
```

---

## ModelSim / Questa (Optional)

For simulation, Vivado's built-in simulator works fine for most purposes. However, for large designs or advanced verification, **ModelSim** (free versions available) or **Questa** (paid) offer:

- Faster simulation speed
- Better debugging tools
- SystemVerilog and UVM support for verification

Vivado integrates with ModelSim — you can use it as the simulator in your project settings.

---

## GTKWave (Waveform Viewer)

You already installed this in Week 3. It reads VCD (Value Change Dump) files generated by iverilog or Vivado simulation. GTKWave is not a simulator — it only views pre-recorded waveform files.

For interactive debugging of larger designs, the Vivado Waveform viewer (built-in) is more convenient.

---

## ILA — Integrated Logic Analyzer

One of the most powerful features in Vivado: the **ILA** (Integrated Logic Analyzer).

The ILA is a soft core you instantiate in your design. It connects to internal signals and samples them in real time. Data is stored in on-chip BRAM, then read out via JTAG to Vivado's waveform viewer — while the FPGA is running in hardware.

```
Your Design:                    Host PC (Vivado)
┌──────────────────────────┐         │
│                          │         │
│  your_module ──── sig1 ──┼──▶ILA ─┼──▶ JTAG ──▶ Waveform viewer
│              ──── sig2 ──┼──▶     │
│              ──── sig3 ──┼──▶     │
└──────────────────────────┘         │
```

This is how you debug hardware that is too fast to observe otherwise. You set a trigger condition (e.g., "capture when error_flag = 1") and the ILA captures hundreds of cycles before and after the trigger.

---

**Next:** [Part 4 — Installing and using Vivado](../part4-installing-vivado/README.md)
