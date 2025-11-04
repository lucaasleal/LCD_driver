# Verilog Module Documentation: LCD Driver

## 1. Overview

This Verilog module (`LCD.v`) was developed as a project for the **Digital Systems (Sistemas Digitais)** course as part of the **Computer Engineering program at the Federal University of Pernambuco (UFPE)**.

It implements a driver for an alphanumeric Liquid Crystal Display (LCD), compatible with the standard HD44780 controller. The module is designed to act as a peripheral for a simple processor, displaying information about the instruction being executed.

It is capable of receiving an `opcode` (operation code), a register address (`endreg`), and an immediate value (`imm`), formatting and displaying this information on a two-line display.

## 2. Key Features

* **LCD Control:** Generates the control signals (`EN_out`, `RW_out`, `RS_out`) and data (`out`) necessary to operate an LCD in 8-bit mode.

* **Finite State Machine (FSM):** Uses an FSM to manage the LCD initialization process and data writing, respecting the required timings (delays).

* **Button Debouncing:** Includes debouncing logic for two input buttons (`start_button` and `send_button`).

* **Output Formatting:** Dynamically displays instruction information in a predefined format:

  * **Line 1:** Shows the operation mnemonic (e.g., "LOAD", "ADD") and a register value (e.g., `[1010]`).

  * **Line 2:** Shows a 7-bit immediate value, formatted as a 5-digit signed decimal number (e.g., `+00042`, `-00015`).

## 3. Interface (Inputs and Outputs)

### Inputs

| Name | Width (bits) | Description | 
 | ----- | ----- | ----- | 
| `clk` | 1 | Main system clock. | 
| `start_button` | 1 | Button to enable/disable the module (controls the `valid` flag). | 
| `send_button` | 1 | Button to control the data display (controls the `on` flag). | 
| `opcode` | 3 | Operation code (e.g., LOAD, ADD, SUB) to be displayed. | 
| `endreg` | 4 | Destination/source register value (displayed in binary). | 
| `imm` | 7 | Immediate value (displayed as signed decimal). | 

### Outputs

| Name | Width (bits) | Description | 
 | ----- | ----- | ----- | 
| `EN_out` | 1 | "Enable" (EN) signal for the LCD. | 
| `RW_out` | 1 | "Read/Write" (R/W) signal for the LCD. (Fixed to 0, "Write" mode). | 
| `RS_out` | 1 | "Register Select" (RS) signal for the LCD. (0 for Commands, 1 for Data). | 
| `out` | 8 | 8-bit data bus for the LCD. | 
| `led1`, `led2` | 1 | Status LEDs (indicate which line is being written to). | 

## 4. LCD Display Format

The module formats the output on two lines:

>+----------------+
>
>|OPCODE [REG]    |  <-- Line 1
>
>|+/-IMMEDIATE    |  <-- Line 2
>
>+----------------+



**Example:**
For `opcode = 3'b001` ("ADD"), `endreg = 4'b1010`, and `imm = 7'b0011001` (25 in decimal):

>+----------------+
>
>|ADD [1010] |
>
>   |+00025 |
>
>+----------------+


## 5. Internal Operation

### Button Logic

1. **`start_button` (Debounce `btstr_state`):**

   * When pressed and released, this button toggles the `valid` register.

   * The main state machine (`always @(posedge clk && valid)`) only advances when `valid` is `1`. This essentially acts as the module's "on/off" switch.

2. **`send_button` (Debounce `btsnd_state`):**

   * When pressed and released, this button toggles the `on` register.

   * The data selection logic (`always @(...) if(~on)`) is only active when `on` is `0`. This suggests the button is used to "freeze" or "unfreeze" the data display on the LCD.

### Main State Machine (`state`)

The main FSM controls the writing sequence to the LCD.

* **`INIT` (State 0):**

  * Initialization state. Sends the standard command sequence to configure the LCD (8-bit mode, 2 lines, clear display, cursor home).

  * Writes the static "template" to the display: `---- [----]` on line 1 and `+00000` on line 2.

  * When finished, it raises the `init_done` flag and transitions to `OPRT` (via `WAIT`).

* **`WAIT` (State 1):**

  * Wait state. Waits for `MS` (50,000) clock cycles. With a 50 MHz clock, this generates a 1ms delay, which is necessary for the LCD to execute commands.

  * Acts as a central "router": based on the `..._done` flags, it decides the next state to execute (`INIT`, `OPRT`, `ENDR`, `DATA`) or if it should remain idle.

* **`OPRT` (State 2):**

  * Writes the operation mnemonic.

  * Uses the input `opcode` to select the correct string (e.g., "LOAD", "ADD", "SUBI").

  * Writes the string to the beginning of line 1.

  * When finished, it raises the `oprt_done` flag and returns to `WAIT`.

* **`ENDR` (State 3):**

  * Writes the register value (`endreg`).

  * Positions the cursor and writes the 4 bits of `endreg` (as '0' or '1') inside the brackets `[ ]` on line 1.

  * If the `opcode` is `3'b110` (CLR), it writes `----` to clear the register info.

  * When finished, it raises the `endr_done` flag and returns to `WAIT`.

* **`DATA` (State 4):**

  * Writes the immediate value (`imm`).

  * Positions the cursor to line 2.

  * Converts the 7-bit value `imm[5:0]` into 5 decimal digits and displays them.

  * Uses `imm[6]` to determine the sign (`+` or `-`).

  * When finished, it raises the `data_done` flag and returns to `WAIT`.

The FSM is designed to execute the sequence `INIT` -> `OPRT` -> `ENDR` -> `DATA`. After `DATA`, it resets the `..._done` flags and stays in `WAIT`, ready to display a new set of data (OPRT -> ENDR -> DATA) when the inputs change.

### Opcode Mapping

The logic in the `OPRT` state maps the 3-bit `opcode` to the following strings:

| `opcode` | String Displayed | 
 | ----- | ----- | 
| `3'b000` | "LOAD" | 
| `3'b001` | "ADD " | 
| `3'b010` | "ADDI" | 
| `3'b011` | "SUB " | 
| `3'b100` | "SUBI" | 
| `3'b101` | "MUL " | 
| `3'b110` | "CLR " | 
| `3'b111` | "DPL " |
