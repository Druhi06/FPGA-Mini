Task 3

[VSDSquadron FPGA Board Mini Project 3.docx](https://github.com/user-attachments/files/19780151/VSDSquadron.FPGA.Board.Mini.Project.3.docx)

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

### 2️⃣ README.md

<details>
<summary>Click to expand README.md description</summary>

*## What does it do?
This example is used to show how to communicate from the FPGA chip to your computer using a USB-C cable with the [UART protocol](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter).
Its sending out the character 'D' repeatedly all the time. If you press any keys on the keyboard, the output won't change because this example is just used to transmit to the PC, not to receive any data from the PC.

## Testing on Windows 
To check it, one should install PuTTY, it is used as a terminal emulator.
It's a completely free, open source sofware.
You can download PuTTY from the URL- https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
To download it from the given URL, just select the Package files from MSI(Windows Installer), depending on the computer you have. 
When you open PuTTY app after it's installed on your machine, just select the connection type as Serial, then you should check which 
COM port is working by taking a look in Device Manager ![WhatsApp Image 2024-12-29 at 02 43 53_3977c1b4](https://github.com/user-attachments/assets/ff155c72-e850-4397-81f0-21e0d7836902)
 of your computer. On our computer COM3 port was assigned to the FPGA board. Then select the required COM port and click open for PuTTY to start. 
Before running PuTTY one must disconnect the FPGA board from the Virtual box. But the FPGA must be physically connected with the computer
with the USB cable.
The ouptut 'D" should appear in the PuTTY like shown in the image below.
![WhatsApp Image 2024-12-28 at 23 53 37_0b86834c](https://github.com/user-attachments/assets/d55256ff-2a23-4e0c-a19f-be0eca765059)

## Testing on Linux terminal
Connect the FPGA board to the Virtual box.
Install picocom in the Linux terminal by running command ```apt install picocom```
Then use the command ```make terminal``` to run the picocom. It will start displaying the output 'D' in the terminal. 
To exit and stop, use ```Ctrl A+X```

## How does it work?
The UART protocol is implemented im the module uart_trx.v file.
It works in one direction only, ie. it sends data without having a provison to receive the data back from the receiver. 
For UART to work, the Baud rate shoud be the same on both the transmitting anf receiving side. Here the Baud rate is 9600 Hz.

</details>

---

### 3️⃣ VSDSquadronFM.pcf

<details>
<summary>Click to expand VSDSquadronFM.pcf description</summary>

set_io  led_green 40
set_io  led_red	39
set_io  led_blue 41
set_io  uarttx 14
set_io  hw_clk 20 

</details>

---

### 4️⃣ top.v

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
    .RGB0PWM (frequency_counter_i[24]&frequency_counter_i[23] ),
    .RGB1PWM (frequency_counter_i[24]&~frequency_counter_i[23]),
    .RGB2PWM (~frequency_counter_i[24]&frequency_counter_i[23]),
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

### 5️⃣ uart_trx.v

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
