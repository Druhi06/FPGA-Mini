Task 4

[VSDSquadron FPGA Mini Board Project 4.docx](https://github.com/user-attachments/files/19780207/VSDSquadron.FPGA.Mini.Board.Project.4.docx)

---

### 1️⃣ Makefile

<details>
<summary>Click to expand Makefile description</summary>

# Define project-specific variables
TOP=top
PCF_FILE=VSDSquadronFM
BOARD_FREQ=12
CPU_FREQ=20
FPGA_VARIANT=up5k
FPGA_PACKAGE=sg48
VERILOG_FILE=top.v

#Uart Var	
PICO_DEVICE=/dev/ttyUSB0   # replace by the terminal used by your device
BAUDS=9600

build:
	yosys -DCPU_FREQ=$(CPU_FREQ) -q -p "synth_ice40 -abc9 -device u -dsp -top $(TOP) -json $(TOP).json" $(VERILOG_FILE)
	nextpnr-ice40 --force --json $(TOP).json --pcf $(PCF_FILE).pcf --asc $(TOP).asc --freq $(BOARD_FREQ) --$(FPGA_VARIANT) --package $(FPGA_PACKAGE) --opt-timing -q
	icetime -p $(PCF_FILE).pcf -P $(FPGA_PACKAGE) -r $(TOP).timings -d $(FPGA_VARIANT) -t $(TOP).asc
	icepack -s $(TOP).asc $(TOP).bin

flash:
	iceprog $(TOP).bin

clean:
	rm -rf $(TOP).blif $(TOP).asc $(TOP).bin $(TOP).json $(TOP).timings

terminal:
	sudo picocom -b $(BAUDS) $(PICO_DEVICE) --imap lfcrlf,crcrlf --omap delbs,crlf --send-cmd "ascii-xfr -s -l 30 -n"

cycle:
	git pull
	make
	sudo make flash 

</details>

---

### 2️⃣ VSDSquadronFM.pcf

<details>
<summary>Click to expand VSDSquadronFM.pcf description</summary>

set_io  led_green 40
set_io  led_red	39
set_io  led_blue 41
set_io  uarttx 14
set_io  uartrx 15
set_io  hw_clk 20

</details>

---

### 3️⃣ top.v

<details>
<summary>Click to expand top.v description</summary>

`include "uart_trx.v"

//----------------------------------------------------------------------------
//                                                                          --
//                         Module Declaration                               --
//                                                                          --
//----------------------------------------------------------------------------
module top (
  // outputs
  output wire led_red  , // Red
  output wire led_blue , // Blue
  output wire led_green , // Green
  output wire uarttx , // UART Transmission pin
  input wire uartrx , // UART Transmission pin
  input wire  hw_clk
);

  wire        int_osc            ;
  reg  [27:0] frequency_counter_i;
  
/* 9600 Hz clock generation (from 12 MHz) */
    reg clk_9600 = 0;
    reg [31:0] cntr_9600 = 32'b0;
    parameter period_9600 = 625;
    
uart_tx_8n1 DanUART (.clk (clk_9600), .txbyte("D"), .senddata(frequency_counter_i[24]), .tx(uarttx));
//----------------------------------------------------------------------------
//                                                                          --
//                       Internal Oscillator                                --
//                                                                          --
//----------------------------------------------------------------------------
  SB_HFOSC #(.CLKHF_DIV ("0b10")) u_SB_HFOSC ( .CLKHFPU(1'b1), .CLKHFEN(1'b1), .CLKHF(int_osc));


//----------------------------------------------------------------------------
//                                                                          --
//                       Counter                                            --
//                                                                          --
//----------------------------------------------------------------------------
  always @(posedge int_osc) begin
    frequency_counter_i <= frequency_counter_i + 1'b1;
        /* generate 9600 Hz clock */
        cntr_9600 <= cntr_9600 + 1;
        if (cntr_9600 == period_9600) begin
            clk_9600 <= ~clk_9600;
            cntr_9600 <= 32'b0;
        end
  end

//----------------------------------------------------------------------------
//                                                                          --
//                       Instantiate RGB primitive                          --
//                                                                          --
//----------------------------------------------------------------------------
  SB_RGBA_DRV RGB_DRIVER (
    .RGBLEDEN(1'b1                                            ),
    .RGB0PWM (uartrx),
    .RGB1PWM (uartrx),
    .RGB2PWM (uartrx),
    .CURREN  (1'b1                                            ),
    .RGB0    (led_green                                       ), //Actual Hardware connection
    .RGB1    (led_blue                                        ),
    .RGB2    (led_red                                         )
  );
  defparam RGB_DRIVER.RGB0_CURRENT = "0b000001";
  defparam RGB_DRIVER.RGB1_CURRENT = "0b000001";
  defparam RGB_DRIVER.RGB2_CURRENT = "0b000001";

endmodule

</details>

---

### 4️⃣ uart_trx.v

<details>
<summary>Click to expand uart_trx.v description</summary>

// 8N1 UART Module, transmit only

module uart_tx_8n1 (
    clk,        // input clock
    txbyte,     // outgoing byte
    senddata,   // trigger tx
    txdone,     // outgoing byte sent
    tx,         // tx wire
    );

    /* Inputs */
    input clk;
    input[7:0] txbyte;
    input senddata;

    /* Outputs */
    output txdone;
    output tx;

    /* Parameters */
    parameter STATE_IDLE=8'd0;
    parameter STATE_STARTTX=8'd1;
    parameter STATE_TXING=8'd2;
    parameter STATE_TXDONE=8'd3;

    /* State variables */
    reg[7:0] state=8'b0;
    reg[7:0] buf_tx=8'b0;
    reg[7:0] bits_sent=8'b0;
    reg txbit=1'b1;
    reg txdone=1'b0;

    /* Wiring */
    assign tx=txbit;

    /* always */
    always @ (posedge clk) begin
        // start sending?
        if (senddata == 1 && state == STATE_IDLE) begin
            state <= STATE_STARTTX;
            buf_tx <= txbyte;
            txdone <= 1'b0;
        end else if (state == STATE_IDLE) begin
            // idle at high
            txbit <= 1'b1;
            txdone <= 1'b0;
        end

        // send start bit (low)
        if (state == STATE_STARTTX) begin
            txbit <= 1'b0;
            state <= STATE_TXING;
        end
        // clock data out
        if (state == STATE_TXING && bits_sent < 8'd8) begin
            txbit <= buf_tx[0];
            buf_tx <= buf_tx>>1;
            bits_sent = bits_sent + 1;
        end else if (state == STATE_TXING) begin
            // send stop bit (high)
            txbit <= 1'b1;
            bits_sent <= 8'b0;
            state <= STATE_TXDONE;
        end

        // tx done
        if (state == STATE_TXDONE) begin
            txdone <= 1'b1;
            state <= STATE_IDLE;
        end

    end

endmodule
</details>

---

### 5️⃣ .gitignore

<details>
<summary>Click to expand .gitignore description</summary>

**/*.blif 
**/*.asc 
**/*.bin 
**/*.json
**/*.timings
**/*.out

</details>

---

### 6️⃣ README.md

<details>
<summary>Click to expand README.md description</summary>

# VSDSquadron_FM

This repository contains projects for VSDSquadron_FM, utilizing open-source FPGA tools for development.

## Toolchain Requirements

Ensure the following tools are installed and configured:

1. **[Project IceStorm](https://github.com/YosysHQ/icestorm)**  
   Toolchain for Lattice iCE40 FPGAs. Install this first.

2. **[Yosys](https://github.com/YosysHQ/yosys)**  
   Open-source synthesis tool.

3. **[nextpnr](https://github.com/YosysHQ/nextpnr)**  
   Open-source Place & Route (P&R) tool.

## Running a Project

Follow these steps to build and flash a project:

1. Navigate to the specific project folder:
   ```bash
   cd <project-folder>
   ```

2. Build the binaries:
   ```bash
   make build
   ```

3. Flash the code to external SRAM:
   ```bash
   make flash
   ```

For cleanup, use:
```bash
make clean
```

---

# List of Example-projects:

1. **[blink_led](blink_led/)**  
    It blinks the led in different colours. It is using an internal oscillator as a time source for blinking.

2. **[blink_hw](blink_hw/)**   
   It blinks the led in different colours by using the crystal hardware osscilator on the board. 

3. **[led_red](led_red/)**  
    Constantly lights up the RGB led with red light.

4. **[led_blue](led_blue/)**  
    Constantly lights up the RGB led with blue light.

5. **[led_green](led_green/)**  
    Constantly lights up the RGB led with green light.

6. **[led_white](led_white/)**  
    Constantly lights up the RGB led with white light.

7. **[uart_tx](uart_tx/)**   
    It sends the 'D' characters repeatedly from the FPGA through USB to the computer. 

8. **[uart_tx_sense](uart_tx_sense/)**  
    It sends the 'D' characters repeatedly from the FPGA through USB to the computer, and lights up the LED whenever a character is received from the PC

9. **[uart_loopback](uart_loopback/)**  
   It just receives the signal directly from the PC to the FPGA. So whatever character we type on the keyboard appears as the output.

10. **[simpleuart](simpleuart/)**   
    Its a controller that accepts commands from the PC keyboard. It parses the input and plays with the output.
_
11. **[ram_infer](ram_infer/)**   
    It is testing the inferences of RAM blocks in the FPGA.

12. **[nandcontroller](nandcontroller/)**   
    Its a controller to interface with NAND Flash memory, for USB pen drives.
    
13. **[RISCV](RISCV/)**   
     Its a CPU based on RISC-V instruction set architecture.

    
Happy hacking! 🚀

</details>

---
