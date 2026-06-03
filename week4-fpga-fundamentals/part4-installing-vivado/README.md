# Part 4 — Installing Vivado

> Vivado is large and the installation has a few steps that trip people up. Follow this carefully.

---

## System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Windows 10 / Ubuntu 18.04 | Windows 11 / Ubuntu 22.04 |
| RAM | 8 GB | 32 GB |
| Disk Space | 70 GB | 100+ GB |
| CPU | Any x86_64 | 8+ cores (faster implementation) |
| Display | 1024×768 | 1920×1080 |

> **Note:** Vivado does not run on ARM (M1/M2 Mac, Raspberry Pi). You need an x86_64 machine, or use a VM/cloud instance.

---

## Step 1: Create an AMD/Xilinx Account

Go to: https://www.xilinx.com/registration/

Register a free account. You will need this to download Vivado.

---

## Step 2: Download the Vivado Installer

1. Go to: https://www.xilinx.com/support/download.html
2. Select **Vivado ML Edition**
3. Under the latest version, download the **Unified Installer** (not the individual component installers)
   - Windows: `FPGAs_AdaptiveSoCs_Unified_<version>_Win64.exe` (~50 MB bootstrapper, downloads the rest during install)
   - Linux: `FPGAs_AdaptiveSoCs_Unified_<version>_Lin64.bin`

The bootstrapper is small; the actual download during installation is ~25–50 GB. **Ensure you have a stable internet connection before starting.**

Alternatively, download the **Full Product Installation Image** if you want a local offline installer (this is the full ~50 GB download upfront).

---

## Step 3: Install (Linux)

```bash
# Make the installer executable
chmod +x FPGAs_AdaptiveSoCs_Unified_*.bin

# Run the installer
./FPGAs_AdaptiveSoCs_Unified_*.bin
```

In the GUI installer:
1. **Select Product to Install:** Choose **Vivado**
2. **Select Edition:** Choose **Vivado ML Standard** (free)
3. **Customization:**
   - Under "Devices", expand "7 Series" → select **Artix-7**, **Kintex-7**, **Zynq-7000** (uncheck others to save space)
   - Under "Installation Options", ensure "Install Cable Drivers" is checked
4. **Installation Directory:** Default is `/tools/Xilinx` — fine to keep
5. Accept the license agreements
6. Click Install → wait 30–90 minutes

### Linux Post-Installation

Install cable drivers (required to program FPGA via USB-JTAG):

```bash
cd /tools/Xilinx/Vivado/<version>/data/xicom/cable_drivers/lin64/install_script/install_drivers
sudo ./install_drivers
```

Add Vivado to your PATH. Add this to `~/.bashrc`:

```bash
source /tools/Xilinx/Vivado/<version>/settings64.sh
```

Then:
```bash
source ~/.bashrc
vivado &    # test that it launches
```

---

## Step 4: Install (Windows)

1. Run the `.exe` installer as Administrator
2. Follow the same GUI steps as Linux above
3. Cable drivers are installed automatically
4. Vivado will appear in the Start menu

---

## Step 5: Verify the Installation

Launch Vivado:

```bash
# Linux
vivado

# Or open Vivado from Start Menu on Windows
```

You should see the Vivado welcome screen. If it launches without errors, the installation is successful.

---

## Step 6: Install Vitis (For Week 6)

Vitis is installed separately from the same unified installer, or can be added during Vivado installation.

**If you did not install Vitis during Vivado installation:**

1. Re-run the installer
2. Select **Vitis** instead of Vivado
3. Or select both if you want them side by side

After installation, add Vitis to your PATH:

```bash
source /tools/Xilinx/Vitis/<version>/settings64.sh
```

---

## Your First Vivado Project — LED Blinker

Let us verify Vivado works by creating the simplest possible FPGA project: blinking an LED.

### The Design

```verilog
// blink.v
// Blinks an LED every 0.5 seconds on a 100 MHz clock

module blink (
    input  wire clk,    // 100 MHz clock
    output reg  led     // LED output
);
    // 100 MHz / 2 = 50,000,000 counts for 0.5 second
    localparam COUNT_MAX = 26'd50_000_000 - 1;
    
    reg [25:0] counter;

    always @(posedge clk) begin
        if (counter == COUNT_MAX) begin
            counter <= 26'd0;
            led     <= ~led;    // toggle LED
        end else begin
            counter <= counter + 1;
        end
    end

endmodule
```

### Creating the Project in Vivado

1. **Open Vivado** → Click "Create Project"
2. **Project Name:** `led_blink`
3. **Project Type:** RTL Project, "Do not specify sources at this time"
4. **Default Part / Boards:** Select your board from the list, or select the FPGA part number directly
   - Nexys A7 100T: `xc7a100tcsg324-1`
   - Basys 3: `xc7a35tcpg236-1`
   - Arty A7-35T: `xc7a35ticsg324-1L`
5. Click Finish

### Add Source File

1. In the Sources panel, right-click "Design Sources" → "Add Sources"
2. "Add or create design sources" → Next
3. "Create File" → name it `blink.v` → Finish
4. When prompted for port definitions, click OK (we already wrote the code)
5. The file opens — paste in the blink.v code above
6. Save (Ctrl+S)

### Add Constraints File

1. In the Sources panel, right-click "Constraints" → "Add Sources"
2. "Add or create constraints" → Next
3. "Create File" → name it `blink.xdc` → Finish

Open blink.xdc and add the constraints for your board.

**For Nexys A7:**
```tcl
# System clock 100 MHz
create_clock -period 10.000 -name sys_clk [get_ports clk]
set_property PACKAGE_PIN E3      [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports clk]

# LED[0]
set_property PACKAGE_PIN H17     [get_ports led]
set_property IOSTANDARD LVCMOS33 [get_ports led]
```

**For Basys 3:**
```tcl
create_clock -period 10.000 -name sys_clk [get_ports clk]
set_property PACKAGE_PIN W5      [get_ports clk]
set_property IOSTANDARD LVCMOS33 [get_ports clk]

set_property PACKAGE_PIN U16     [get_ports led]
set_property IOSTANDARD LVCMOS33 [get_ports led]
```

### Run Synthesis and Implementation

1. In the Flow Navigator (left panel), click **Run Synthesis** → OK
2. When synthesis finishes, click **Run Implementation** → OK
3. When implementation finishes, click **Generate Bitstream** → OK

The full flow takes 2–5 minutes for this simple design.

### Program the FPGA

1. Connect your FPGA board to your computer via USB
2. Power on the board
3. In Vivado: **Open Hardware Manager** → **Open Target** → **Auto Connect**
4. Click **Program Device**
5. The bitstream file path should be auto-filled → Click **Program**

After programming, the LED should blink approximately once per second. If it does, your entire toolchain is working correctly.

---

## Troubleshooting Common Issues

### "No hardware target found"

- Board not connected, or not powered on
- Cable drivers not installed (Linux: re-run the cable driver installer)
- USB cable issue — try a different cable or port
- On Linux, add yourself to the `dialout` group: `sudo adduser $USER dialout` then log out and back in

### Synthesis fails with "top module not found"

- The module name in your Verilog (`module blink`) must match the "Top Module" in the synthesis settings. Check Project Settings → Synthesis → Top.

### "Timing constraints not met" after implementation

- For the blink example this should not happen (the design is trivial)
- For more complex designs, this is a normal issue — covered in Week 7

### Vivado is very slow

- Implementation is CPU-intensive. Ensure other applications are closed.
- In synthesis/implementation settings, set "Number of jobs" to match your CPU core count.
- For repeated design changes, use "incremental implementation" which only re-routes changed portions.

---

**Next:** [Part 5 — FPGA Internals Deep: Configuration, Routing, IOB, Reconfiguration](../part5-fpga-internals-deep/README.md)
