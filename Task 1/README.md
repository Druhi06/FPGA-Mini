Task 1

[VSDSquadron FPGA Mini Board Project.docx](https://github.com/user-attachments/files/19779887/VSDSquadron.FPGA.Mini.Board.Project.docx)

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
BAUDS=115200

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

set_io  led_red	39
set_io  led_blue 40
set_io  led_green 41
set_io  hw_clk 20
set_io  testwire 17

</details>

---

### 3️⃣ top.v

<details>
<summary>Click to expand top.v description</summary>

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
  input wire hw_clk,  // Hardware Oscillator, not the internal oscillator
  output wire testwire
);

  wire        int_osc            ;
  reg  [27:0] frequency_counter_i;

  assign testwire = frequency_counter_i[5];
 
  always @(posedge int_osc) begin
    frequency_counter_i <= frequency_counter_i + 1'b1;
  end


//----------------------------------------------------------------------------
//                                                                          --
//                       Counter                                            --
//                                                                          --
//----------------------------------------------------------------------------

//----------------------------------------------------------------------------
//                                                                          --
//                       Internal Oscillator                                --
//                                                                          --
//----------------------------------------------------------------------------
  SB_HFOSC #(.CLKHF_DIV ("0b10")) u_SB_HFOSC ( .CLKHFPU(1'b1), .CLKHFEN(1'b1), .CLKHF(int_osc));


//----------------------------------------------------------------------------
//                                                                          --
//                       Instantiate RGB primitive                          --
//                                                                          --
//----------------------------------------------------------------------------
  SB_RGBA_DRV RGB_DRIVER (
    .RGBLEDEN(1'b1                                            ),
    .RGB0PWM (1'b0), // red
    .RGB1PWM (1'b0), // green
    .RGB2PWM (1'b1), // blue
    .CURREN  (1'b1                                            ),
    .RGB0    (led_red                                       ), //Actual Hardware connection
    .RGB1    (led_green                                       ),
    .RGB2    (led_blue                                        )
  );
  defparam RGB_DRIVER.RGB0_CURRENT = "0b000001";
  defparam RGB_DRIVER.RGB1_CURRENT = "0b000001";
  defparam RGB_DRIVER.RGB2_CURRENT = "0b000001";

endmodule

</details>

---
