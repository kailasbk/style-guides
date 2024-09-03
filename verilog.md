# (System)Verilog Style Guide

The following is a style guide for Verilog and SystemVerilog, especially for
the synthesis constructs of the languages.

## Alignment

Tab size of 2 spaces should be used throughout for indentation. Verilog code
can become very tab heavy at times, so a tab size of 4 will use up a lot of
space.

## Signals

In this document, all 4-state objects used for synthesis will be referred to as
'signals' for simplicity sake and to avoid the confusion of variables and nets
that existed with Verilog and because the term 'object,' which is used in the
SystemVerilog spec, may be confusing for people coming from OOP backgrounds.

### `logic` versus `wire` and `reg`

For all signals, use the SystemVerilog `logic` type instead of the old Verilog
`wire` and `reg` types unless using the `wire` is required (e.g. for multiple
drivers). Given this, the terms 'signals' and 'logics' can basically be used
synonymously.

### Naming

All signal names should be in snake case (e.g `uart_baud_counter`).

Signal names should also follow the following conventions:
- For signals without the typical active high usage, suffixes should be added
  for clarity.
  - For active low signals, the `_n` suffix should be added (e.g. `rst_n`).
  - For differential pairs, the `_p` suffix should be added to the active high
    signal while the `_n` suffix should be added to the active low signal (e.g.
    `sync_p` and `sync_n`).
- Names should generally be descriptive, so avoid names like `a` and `value`.
  - For handshake signals, generally keep to the `valid` and `ready` signal
    names, unless the semantics of the handshake differ from the typical AXI4
    handshake.
  - Boolean-like signals should be named either after what they represent (e.g.
    `is_about_to_overflow` or `can_decode`) or the action that occurs when the
    signal goes high (e.g. `clear_fifo` or `increment_ptr`).

### Declaration

Some tools (like Vivado) will allow signals to be declared anywhere in the
scope that they are used, but non-port signals should generally be declared
shortly before their first reference.

Related signals should be tabularly aligned, with spaces, with alignment being
towards the right (i.e. towards the name instead of the `logic` keyword). There
should be a space after the `logic` key and before the signal name, but not in
between the dimensions, like so:

```sv
// right aligned
logic             bus_valid;
logic             bus_ready;
logic       [7:0] bus_strb;
logic      [31:0] bus_addr;
logic [3:0][63:0] bus_data;
```

But *not* like so:

```sv
// left aligned
logic             bus_valid;
logic             bus_ready;
logic [7:0]       bus_strb;
logic [31:0]      bus_addr;
logic [3:0][63:0] bus_data;
```

The reasons for right alignment are as follows:
- Almost all logic array declarations have `0` as the LSB, so the LSB portions
  of the declarations align.
- When array sizes in an aligned block differ, it is easier to read if the
  array dimensions are next to the signal names (you already know that it is
  type `logic`).

Using spaces instead of tabs for alignment may seem tedious, but with tab set
to 2 spaces, the space key only needs to be used at most once per line.

### Reuse

Just like the debate of when to package a section of code into a function in
programming, a similar issue can arise with signals in HDLs. A value can be
referenced as a combination of existing signals or packaged into a new signal
with a new name, and when do to so can be a subjective matter.

Luckily, for port signals, the interface is generally set, and thus the ports
simply match to that specification.

However, for internal module signals, when to instantiate a new signal is less
objective. This guide suggests the following situations for when to make a new
signal:
- A given function of two or more signals is used more than once in the module.
  For example, the expression `!full && res_valid && no_error` is used on 3
  seperate occassions in the module, so making a new signal `store_result`
  would be more concise and reduce the chance of typos.
- The signal is a portion of a larger signal such that the name of the array is
  not adequately descriptive of the child signals. For example, a packed array
  `coords[29:0]` contains `x[9:0]`, `y[9:0]` and `z[9:0]`. Making the three new
  signals add clarity to what they represent.

Of course, if a signal is needed to represent some storage or logic, make that
signal; this guidance is just for signals that are aliases for existing logic
or signals.

## Operators, expressions, and statements

In general, as

### Logical operators

For creating boolean logic, or logic that writes a 1-bit signal that is
interpreted as a true or false value, the logical operators should be used
instead of the bitwise operators.

More concretely, use `||`, `&&` and `!` over their respective bitwise
counterparts, being `|`, `&`, and `!`.

Do not rely on implicit casting down to 1-bit from more than 1-bit signals.
This may work in languages like C, but the width conversion semantics in
Verilog are not the same.

```sv
logic [3:0] count;
logic       not_empty;

assign not_empty = count; // DO NOT do this
assign not_empty = count != 4'b0; // DO this instead
```

### Arithmetic operators

#### Signed arithmetic

Make note that an arithmetic expression in SystemVerilog is not signed unless
*all of the operands* are signed. When casting from a unsigned to signed value,
using the `$signed()` built-in function, which is supported by both Verilog and
SystemVerilog, is recommended. Remember to add padding to the MSB of the value
so that proper sign extension occurs (the sign bit of an unsigned value should
always be 0).

```sv
logic signed [4:0] old_pos;
logic signed [4:0] new_pos;
logic        [2:0] left_move;
logic        [2:0] right_move;

// BUG: this will not do signed arithmetic
assign new_pos = old_pos - left_move + right_move;
// BUG: this will not sign-extend properly
assign new_pos = old_pos - $signed(left_move) + $signed(right_move);
// adding padding properly sign-extends
assign new_pos = old_pos - $signed({1'b0, left_move}) + $signed({1'b0, right_move});
```

### Sequential block statements (`begin` and `end`)

There is some leeway with the placement of `begin` and `end` for sequential
block statements. However, placement should be space efficient and avoid common
bugs.

There are four universal rules that can be applied to begin-end statements:
- Any section of code that does not reside on the same line as its parent
  should be enclosed in a begin-end statement. It is possible to skip the block
  statement in some cases, like a single line after an `if` or an if-else after
  an `always_ff`, but this can lead to bugs later when modifications are made.
- The `begin` statement should *always* be on the same line as the block's
  parent statement, whether it be if-else, a process, a case element, etc.
- Remember that `module`, `case`, and `function` do not use block statements
  and are instead closed with `endmodule`, `endcase`, and `endfunction`.
  Especially with `case`, it can be easy to mistakenly use `begin` and `end`
  and get an unexpected syntax error.
- Sequential block statements should never be used by themselves in
  synthesizable code. This can have some use when combined with fork-join
  parallel block statements in simulation code, but serves no purpose in RTL.

The placement of the `end` keyword is somewhat up to personal taste, like in
the following examples.

```sv
// this is the most compact
if (cond) begin
  // some stuff
end else begin
  // other stuff
end

// this is less compact, but reasonable
if (cond) begin
  // some stuff
end
else begin
  // other stuff
end

// this is wasting lines
if (cond)
begin
  // some stuff
end
else
begin
  // other stuff
end

// this is just weird
if (cond)
begin
  // some stuff
end else
begin
  // other stuff
end

// c'mon...
if (cond)
begin
  // some stuff
end else begin
  // other stuff
end
```

### `if`/`else if`/`else` statements

The `begin` and `end` *can*, but do not need to be, omitted in the case of one
line statements. However, do not place statements on the next line without a
begin-end block, as if subsequent statements are added later, they will not
properly nest, leading to bugs. For example, see the following cases:

```sv
// this is OK
if (clear_counter) counter = 1'b0;
else               counter = old_counter;

// this is also OK
if (clear_counter) begin
  counter = 1'b0;
end else begin
  counter = old_counter;
end

// don't do this, can lead to bugs later
if (clear_counter)
  counter = 1'b0;
else
  counter = old_counter;
```

It should be noted, however, that cases like the above, where a the value of a
single variable is set based on a set of conditions, can also be written with a
single ternary statement. In the above case, it would look like:

```sv
counter = (clear_counter) ? 1'b0 : old_counter;

// can also work for more conditions
counter = clear_counter ? 1'b0               :
          inc_counter   ? old_counter + 1'b1 :
                          old_counter        ;
```

### `case` statements

Case statements are often used as a more consise alternative to if-else
statements, and should be used whenever the logic occurs depending on the value
of a single variable. Each case should have no space before the colon and a
space after the colon. Begin-end blocks are required if a case has more than
one statement, but can be skipped otherwise. These practices can be seen in the
following state machine example:

```sv
// can write this with an if/else statement
if (state == Idle) begin
  // logic for Idle state
end else if (state == Receive) begin
  // logic for Receive state
end else if (state == Send) begin
  // logic for Send state
end

// but cleaner to write with a case statement
unique case (state)
  Idle: begin
    // logic for Idle state
  end
  Receive: begin
    // logic for Receive state
  end
  State: begin
    // logic for Send state
  end
endcase
```

Or with cases like decoders:

```sv
if (inst[5:2] == ADD)        data = a + b;
else if (inst[5:2] == STORE) data = a;
else if (inst[5:2] == LOAD)  data = c;
else                         data = '0;

// cleaner to write with a case statement
unique case (inst[5:2])
  ADD:     data = a + b;
  STORE:   data = a;
  LOAD:    data = c;
  default: data = '0;
endcase
```

Case statements should not be used with there are only two options, such as the
following simple if-else statement:

```sv
// no need to use a case statement here
if (rec_ready && snd_valid) begin
  deq_ptr = 1'b1;
  data_o = fifo_data;
end else begin
  deq_ptr = 1'b0;
  data_o = '0;
end
```

Only use defaults when that is the true desired behavior (i.e. if you would
have written an else statement), and the `default` case should always be
written as the last case in the list. Thus, do not write empty default cases,
as that defeats the purpose of the `unique` case statement. If you write a
`default` case on a `unique case` statement that should exclusive, it defeats
the purpose of using `unique` in the first place. Thus, there are three
variants of the case statement that are appropriate to use:
- A `unique case` statement without a `default` where it is expected that there
  will always be at least one case which matches. This is generally the case
  for state machines and easily flags states with missing implementations.
- A `unique case` statement with a `default` that sets the signals to their
  default values when none of the cases match (which is expected to happen
  sometimes), like the decoder example above.
- A `case` statement without a `default` where the signals altered by the case
  statement have already been set before the case statement.

Again, to choose between the second and third option, which accomplish similar
logic, defer to how the logic would be best implemented with if-statements. If
an else statement would have been used, use the former, and if not, use the
latter.

## Modules

### Headers

Modules headers should be written in the ANSI style with the name in snake case
(e.g. `super_cool_decoder`), and with a space between the module name and
parameter `#` and between the closing parentheses of the parameters and the
opening parentheses of the ports, and wrapped in explicit declaration guards,
like the following:

```sv
`default_nettype none

module super_cool_decoder #(
  // parameters here
) (
  // ports here
);

  // module body
  // logics, submodules, processes here

endmodule

`default_nettype wire
```

In the case that package contents must be imported to be used in the module
parameters or ports, the following is acceptable:

```sv
module super_cool_decoder
  import need_for_ports_pkg::*;
#(
  // parameters here
) (
  // ports here
);
```

For consistency sake, this means that this style (with a newline between the
module name and the parens), can also be used when there are no imports:

```sv
module super_cool_decoder
#(
  // parameters here
) (
  // ports here
);
```

Modules without parameters can omit the parameter list, like the following:

```sv
module super_cool_decoder (
  // ports here
);
```

```sv
module super_cool_decoder
(
  // ports here
);
```

### Ports

All ports should be declared with an explicit `input`, `output`, or `inout`
directionality, and should be type `logic` unless they are an input, in which
case they should be type `wire` (with `default_nettype none`, `logic` can't be
used for inputs to preserve compatibility between Verilog and SystemVerilog
:|).

As a minimal guard against confusion of similarly named signals, and so that
directionality can be understood without referencing the port declaration, port
names should have the following suffixes denoting their directionality:
- Input ports should have the `_i` suffix.
- Output ports should have the `_o` suffix.
- Bidirectional (`inout`) ports should have the `_io` suffix.
- The directionality suffix should come after `_n` or `_p` suffixes, but
  without another underscore (e.g. `rst_ni` for an active low input or
  `data_po` and `data_no` for differential output)

All related ports should tabularly aligned together. For example, the clock and
reset signals can be aligned together while the signals comprising an AXI4
interface together seperately. Like before, alignment should be to the right.

`input wire` ports should have extra spaces to align with `output logic` ports,
like the following:

```sv
module super_cool_decoder
(
  // clock and reset
  input  wire  clk_i,
  input  wire  rst_ni,
  // input interface
  input  wire        valid_i,
  output logic       ready_o,
  input  wire  [4:0] a_i,
  input  wire  [4:0] b_i,
  // output interface
  output logic       valid_o,
  input  wire        ready_i,
  output logic [5:0] c_o,
);
```

### Parameters

All module parameters should be declared in the parameter list in the module
header. Use upper camel case for parameter names (e.g `LogFifoDepth`).

Any parameters that should not be directly set at instantiation should be set
as local parameters with the `localparam` keyword. Local parameters should
generally be declared at the beginning of the module body.

For local parameters whose values are computed based on the values of
parameters and then used in the port declarations, they should be placed at the
end of the parameter list in the module declaration.

The following shows these rules in practice:

```sv
module resizable_cache #(
  parameter TagSize = 13,
  parameter IndexSize = 3,
  localparam AddrSize = TagSize + IndexSize
) (
  // some ports here
  input  wire                 valid_i,
  output logic                ready_o,
  input  wire  [AddrSize-1:0] addr_i,
  // more ports here
);

  localparam RAMSize = 1 << IndexSize;

  // rest of the module body

endmodule
```

### Continuous assignments (`assign`)

Continuous assignments, made with `assign`, should be considered the default
option for combinational logic.

Examples of logic well suited to be written with said statements include the
following:

```sv
// switching from active low to high (or vice versa)
assign rst = !rst_n;
// destructuring a bus (tabularly align, start from MSB)
assign id    = bus[23:20];
assign count = bus[19:12];
assign data  = bus[11: 0];
// structuring a bus (tabularly align, start from MSB)
assign bus[23:20] = id;
assign bus[19:12] = count;
assign bus[11: 0] = data;
// ternary expressions for simple muxes
assign operand_a = use_immediate ? imm : register_a;
// or priority muxes
assign input_data = src_a_valid ? src_a_data :
                    src_b_valid ? src_b_data : '0;
// boolean expressions
assign can_issue = inst_valid && regs_ready && !execute_stall;
// arithmetic expressions
assign counter_inc = counter + 1'b1;
```

Continous assignments should not be used when it would make more sense to use
an `if` or `case` statement to describe logic, like when multiple signals are
set based on a given condition, or when logical precedence is not easy to
descibe with with ternary and boolean expressions.

### Procedures

Do not use the older `always` procedures from Verilog. Instead, use the
`always_comb` procedure for combinational logic, `always_ff` for sequential
logic that uses flip-flops, and `always_latch` for latched logic.

Latches should generally be avoided as they can make meeting timing constraints
difficult for EDA tools and similar logic can be written with FFs.

#### `always_comb` blocks

Outside of continous assignments, combinational logic can also be written using
`always_comb` blocks. Given that combinational logic should be viewed as
updating immediately (in synthesized reality there is a delay, but the main
point being that the value does not wait for other events to change),
`always_comb` blocks should include exclusively blocking assignments (made with
the `=` operator).

In order to assure that an `always_comb` does not cause the creation of a
latch, each logic written to by the `always_comb` must also have a default
value set at the beginning of the process (which is not dependent on the value
of the logic).

For example, the following would cause latch to be inferred:

```sv
always_comb begin
  if (change_value) begin
    value = 3'd5;
  end
end
```

This *would not* fix the issue:

```sv
always_comb begin
  value = value;

  if (change_value) begin
    value = 3'd5;
  end
end
```

But, this *would* fix the issue:

```sv
always_comb begin
  value = 3'd0;

  if (change_value) begin
    value = 3'd5;
  end
end
```

#### `always_ff` blocks

Sequential flip-flip logic is written using `always_ff` blocks. The general
format of an `always_ff` procedure should look something like the following:

```sv
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    value <= 3'd0;
  end else begin
    value <= new_value;
  end
end
```

Given that FFs update on the clock, this must be listed in the procedure's
sensitivity list. If an asynchronous reset is desired (more common in ASIC than
in FPGA), it should be in the list; if a synchronous reset is desired, it
should not be in the list. Always put a space before and after the sensitivity
list (before `@`, after `)`). Also note the usage of `or` for combining events
in the list. *Do not* use `|` or `||` in the list, as they have a different
meaning and will result in unexpected behavior.

```sv
// active low synchronous reset
always_ff @(posedge clk) begin
  if (!rst_n) begin

// active low asynchronous reset
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin

// active high synchronous reset
always_ff @(posedge clk) begin
  if (rst) begin

// active high asynchronous reset
always_ff @(posedge clk or posedge rst) begin
  if (rst) begin
```

In implementation, FFs can also have an enable. It can generally be left up to
the synthesis tools to decide how to use the enable, but if more explicit
control is desired, this can be done with something like the following:

```sv
// note that `en` is not added to the sensitivity list
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    value <= 3'd0;
  end else if (en) begin
    value <= new_value;
  end
end
```

In some more rare cases, logic without a reset or clock enable may desired. In
this case, the procedure should look like the following, with the reset *never*
in the sensitivity list:

```sv
always_ff @(posedge clk) begin
  value <= new_value;
end
```

Sequential logic is supposed to wait for the clock edge to take new values,
with only values from before the edge impacting the values after the edge, so
only nonblocking assignments (those made with the `<=` operator) should be used
in `always_ff` blocks.

If the blocking `=` operator is used, it is possible for this separation to be
broken, like in the following example:

```sv
logic [2:0] counter;
logic       overflowed;

always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin
    counter = 3'd0;
  end else begin
    counter = counter + 1;
    overflowed = counter == 3'd7;
  end
end
```

In the above case, the incremented value of `counter` is seen by the next line,
so `overflowed` becomes `1` the cycle before the overflow actually occurs (when
incrementing from `3'd7` to `3'd0`).

There may be a temptation to mix blocking and nonblocking assignments in
`always_ff` blocks, but to ensure clarity and correctness, if is better to
instead separate logic into two seperate complementary blocks, which will be
discussed in the following section.

### Complementary procedures

There are sometimes cases where the easiest expression of the desired logic
would have multiple assignments to a signal within a clock period with the
intermediate values being used to trigger other logic. In such cases, a pair of
complementary `always_comb` and `always_ff` blocks should be used, with the
starting values coming from the FFs, going through the combination logic, and
then being written back to the FFs.

For clarity, the signals output by the combinational logic (and doing into the
FFs) should be suffixed with `_d`, and the signals at the output of the FFs
(and going into the combinational logic), should be suffixed with `_q`. If a
module output is directly written by the combinational logic, it does not need
the `_d` suffix, just the `_o` one. If a value in the registers is directly
output, it should be assigned to the `_o` suffixed output signal with a
continuous assignment. The `d` and `q` nomenclature was chosen because it
mimics the common names for the input and output port of a flip-flop.

This technique can be useful for both pipeline stages and for finite-state
machines, as shown in the following examples.

```sv
// Example 1: FSM
state_e state_d, state_q;
logic        mem_valid_d, mem_valid_q;
logic        mem_write_d, mem_write_q;
logic [31:0] mem_addr_d, mem_addr_q;
logic [31:0] mem_data_d, mem_data_q;
logic        mem_ready_d, mem_ready_q;

logic [6:0] opcode;
logic is_store;
logic is_load;

always_comb begin
  state_d = state_q;
  mem_valid_d = mem_valid_q;
  mem_write_d = mem_write_q;
  mem_addr_d  = mem_addr_q;
  mem_data_d  = mem_data_q;
  mem_ready_d = mem_ready_q;
  stall_o = 1'b1;
  data_o  = '0;

  opcode = inst_i[6:0];
  is_store = opcode == OP_STORE;
  is_load  = opcode == OP_LOAD;

  unique case (state_q)
    Idle: begin
      if (valid_i && (is_store || is_load)) begin
        state_d = WaitMemReady;
        mem_valid_d = 1'b1;
        mem_write_d = is_store;
        mem_addr_d  = rvals_i[0] + imm_i;
        mem_data_d  = rvals_i[1];
      end else begin
        stall_o = 1'b0;
      end
    end
    WaitMemReady: begin
      if (mem_ready_i) begin
        mem_valid_d = 1'b0;
        if (is_store) begin
          state_d = Idle;
          stall_o = 1'b0;
        end else begin
          state_d = WaitMemValid;
          mem_ready_d = 1'b1;
        end
      end
    end
    WaitMemValid: begin
      if (mem_valid_i) begin
        state_d = Idle;
        mem_ready_d = 1'b0;
        stall_o = 1'b0;
        data_o  = mem_data_i;
      end
    end
  endcase
end

always_ff @(posedge clk_i or negedge rst_ni) begin
  if (!rst_ni) begin
    state_q <= Idle;
    mem_valid_q <= 1'b0;
    mem_write_q <= 1'b0;
    mem_addr_q  <= 32'h0;
    mem_data_q  <= '0;
    mem_ready_q <= 1'b0;
  end else begin
    state_q <= state_d;
    mem_valid_q <= mem_valid_d;
    mem_write_q <= mem_write_d;
    mem_addr_q  <= mem_addr_d;
    mem_data_q  <= mem_data_d;
    mem_ready_q <= mem_ready_q;
  end
end

assign mem_valid_o = mem_valid_q;
assign mem_write_o = mem_write_q;
assign mem_addr_o  = mem_addr_q;
assign mem_data_o  = mem_data_q;
assign mem_ready_o = mem_ready_q;
```

```sv
// Example 2: Pipeline stage
logic        iss_valid_d, iss_valid_q;
logic [31:0] iss_pc_d, iss_pc_q;
logic [31:0] iss_inst_d, iss_inst_q;

always_comb begin
  iss_valid_d = iss_valid_q;
  iss_pc_d    = iss_pc_q;
  iss_inst_d  = iss_inst_q;
  mem_ready_o = 1'b0;
  fifo_pop = 1'b0;

  if (iss_ready_i) begin
    iss_valid_d = 1'b0;
  end

  if (!iss_valid_d && !fifo_empty) begin
    mem_ready_o = 1'b1;
  end

  if (mem_ready_o && mem_valid_i) begin
    iss_valid_d = epoch_q == fifo_epoch_o;
    iss_pc_d    = fifo_pc_o;
    iss_inst_d  = mem_data_i;
    fifo_pop = 1'b1;
  end

  if (flush_i) begin
    iss_valid_d = 1'b0;
  end
end

always_ff @(posedge clk_i or negedge rst_ni) begin
  if (!rst_ni) begin
    iss_valid_q <= 1'b0;
    iss_pc_q    <= '0;
    iss_inst_q  <= '0;
  end else begin
    iss_valid_q <= iss_valid_d;
    iss_pc_q    <= iss_pc_d;
    iss_inst_q  <= iss_inst_d;
  end
end

assign iss_valid_o = iss_valid_q;
assign iss_pc_o    = iss_pc_q;
assign iss_inst_o  = iss_inst_q;
```

### Instantations

Aside from the top-level module in a design, all module are instantiated within
another module in the design, so it is also useful to have guidelines for said
instantiations.
