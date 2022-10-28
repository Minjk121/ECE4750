# ECE4750 Computer Networks DIS Notes

## DIS SEC03 NOTE
- Ad-Hoc vs. Assertion Testing
    - Ad-hoc is not automatic. & systematic
    - Assert in middle of the code
- Directed vs. Random Testing
- Black-Box vs. White-Box Testing
    - Black - only knowing the interface & specs
    - White - knowing implementation + specs
- Value vs. Delay Testing
- Unit vs. Integration Testing
    - Narrowed scope  = unit -> quicker
- Reference Models

- Overflow
    - 32b - 32b = lower 32 bits shown
    - In lab1, we are generating tinyRV1 MUL which produce the result with throwing away most significant 32 bits -> don’t care yet..
- Pytest
    - Use options 
        - -v: verbose
        - -s: line tracing
- imul/test
    - All the pytest codes
    - V1b, v1c = vector testing 
        - C = parametrized!
Latency insensitive microcontrol
- Fives sink the answer before the sink calculates from source
- Then source calc and check with the answer from the source

Random testing
- How do we know the result of the random input?
- Hard when the output is not predictable like network IP address
- How do we know if we have enough?
    - Think about path / (bit) toggle/functional COVERAGEs 

Delay testing is important
- Especially for the latency insensitive code
- Delaying the message from the source… etc


## DIS SEC05 NOTE - bug hunt

10-step Systematic Debugging Process

Here is our recommended systematic debugging process. Use this process after you have fixed any Verilog syntax errors and you are now getting some kind of incorrect result. REMEMBER: You must use --tb=short or --tb=long to see the error message when using pytest!

### Step 1: Run all tests

Run all of the tests to get a high-level view of which test cases are passing and failing.
```
pytest ../lab2_proc
```

### Step 2: Zoom in on one failing test script

Pick one failing test script to focus on, and run just that test script in isolation with the --verbose flag to get a list of the test cases. Pick the most basic test script that is failing. Always focus on any test cases that are failing on FL model (e.g., ProcFL) first since this means it is a bad test case.

Run specific test script with -v or --verbose
```
pytest ../lab2_proc/test/Proc_Simple.py --verbose
```

### Step 3: Zoom in on one failing test case

Pick one failing test case to focus on, and run just that test case in isolation using -k (or maybe -x). Pick the most basic test case that is failing. Use -s to see the line trace. Use --tb=long or --tb=short to see the error message.

```
pytest ../lab2_proc/test/Proc_Simple.py --verbose -k test_csr[core_stats-gen_core_stats_test] -s --tb=short
```
The result shows: F-D-X-M-W, instruction stage, data memory in a diagram.

### Step 4: Determine the observable error

Look at the line trace and the error message. Determine what is the observable error. Often this will be a stream sink error but it could be some other kind of error. Being able to crisply state the observeable error is critical. Simply saying “my code doesn’t work,” or “my code fails this test case” is not sufficient!

In the example, the memory is 000000, not 1 in a diagram

Add print statement in a python script -> it runs with assembly code.

### Step 5: Confirm the test case is valid

Look at the actual test case (in lab 2 this means look at the assembly sequence). Make absolutely sure you know what the test case is testing and that the test case is valid. You have no hope of debugging your design if you do not understand what correct execution you expect to happen! You might want to run the test case on FL model (e.g., ProcFL) just verify that this actually a valid test case before continuing, although hopefully you spotted any failures on the FL model in step 1.

### Step 6: Work backwards from observable error in line trace to buggy cycle

Work backwards from the observable error on the line trace trying to see what is going wrong from just the line trace. NOTE: In lab 2, you can see the instruction memory request and response and the data memory request and response in the line trace – you can often spot errors for LW or SW right from the line trace by looking at the data memory request and response messages (incorrect message type? incorrect address? incorrect data being read/written?). Similarly you can often spot errors for instruction fetch from the line trace. You can also often see errors in control flow (are the wrong instructions being executed, squashed) or errors in stalling/bypassing logic (is an instruction not stalling when it should?) right from the line trace. You need to work backwards from the observable error to narrow your focus on what part of the design might have a bug (the datapath? the control unit?). Try to narrow your focus to a specific buggy cycle where something is going wrong.

Based on the narrowed focus from step 6, take a quick look at the corresponding code. Check for errors in bitwidth, in signal naming, or in connectivity. If you cannot spot anything obvious then go to the next step. If you spot something obvious skip to step 9.

### Step 7: Zoom in on the buggy cycle in the waveform

Use the --dump-vcd option to generate a VCD file. Open the VCD file in gtkwave. Add the clock, reset, and key signals (e.g., inst_D, inst_X, inst_M, inst_W) to the waveform view. Use the narrowed focus from Step 6 to zoom in on a specific cycle and a specific part of the design where you can clearly see a specific signal that is incorrect.

Zoom in:
- if failing [value_...], then it has an error in computing with values (less coverage?)
- 000000 memory output at line 15 -> look backwards from line 15
    - look for harzards (RAW or WAW) 


### Step 8: Work backwards in space on the buggy cycle in waveform

Work backwards from the signal which is incorrect. Work backwards in the datapath – keep working backwards component by component. For each component look at the inputs (all inputs, look at data inputs and control signals) and look at the outputs (all outputs, look at data outputs and status signals). Check for one of three things:

- (1) are the inputs incorrect and the outputs incorrect for this component? if so you need to continue working backwards – if the incorrect input is a control signal then you need to start working backwards into the control unit;

- (2) are the inputs correct and the outputs incorrect for this component? if so then you have narrowed the bug to be inside the component (maybe it is a bug in the ALU? maybe it is a bug in some other module?); or

- (3) are the inputs correct and the outputs correct for this component? Then you have gone backwards too far and you need to go forward in again to find a signal which is incorrect.

### Step 9: Make a hypothesis on what is wrong and on buggy cycle

Once you find a bug, make a hypothesis about what should happen if you fix the bug. Your hypothesis should not just be “fixing the bug will make the test pass.” It should instead be something like “fixing this bug should make this specific signal be 1 instead of 0” or “fixing this bug should make this specific instruction in the line trace stall”.

If there is a RAW hazard, look for stalling/bypassing logics.

### Step 10: Fix bug and test hypothesis

Fix the bug and see what happens by looking at the line trace and/or waveform. Don’t just see if it passes the test – literally check the line trace and/or waveform and see if the behavior confirms the line trace. One of four things will happen:

- (1) the test will pass and the linetrace/waveform behavior will match your hypothesis – bug fixed!

- (2) the test will fail and the linetrace/waveform will not match your hypothesis – you need to keep working – your bug fix did not do what it was supposed to, and it did not fix the error – undo the bug fix and go back to step 6.

- (3) the test will fail but the linetrace/waveform will match your hypothesis – this means your bug fix did what you expected but there might be another bug still causing trouble – you need to keep working – go back to step 6.

- (4) the test will pass and the linetrace/waveform will not match your hypothesis – you need to keep working – your bug fix did not do what you thought it would even though it cause the test to pass – there might be something subtle going on – go back to step 6 to figure out why the bug fix did not do what you thought it would.

There can be multiple bugs! 

### More tools

```
gtkwave # shows the waveform
```
If stalling, ostall_D / ostall_waadr_X_rsl_D signals should go 1. 

If rs_en signal is 0, then we know the input is wrong.

ZOOM in & make hypothesis & test & Zoom out

```
--dump_vcd
```
Note a couple things about this systematic 10 step process. First, it is a systematic process … it does not involve randomly trying things. Second, the process uses all tools at your disposable: output from pytest, traceback, line tracing, and VCD waveforms. You really need to use all of these tools. If you use line tracing but never use VCD waveforms or you use VCD waveforms and never use line tracing then you are putting yourself at a disadvantage. Third, the process requires you to think critically and make a hypothesis about what should change – do not just change something, pass the test, and move on – change something and see if the line trace and waveforms change in the way you expect. Otherwise you can actually introduce more bugs even though you think are fixing things.

# DIS SEC08 NOTE - [Lab 3, Cache](https://cornell-ece4750.github.io/ece4750-sec08-mem/)

- Modular design (4 registers)
Cache is not combinational, least one edge!
idx: which entry way in the tag we should write

write_init: initializes tag + data array to avoid a compulsary miss, write 4 bytes in data array
-> only for testing!
-> forces data to the cache

- Testing

functional level model simple test:
```
pytest ../lab3_mem/test/simple_test.py -s
```
test simple RTL model from FL model by changing one line:
```
model = TestHarness( CacheFL(), msgs[::2], msgs[1::2] ) # before

model = TestHarness( CacheSimple(), msgs[::2], msgs[1::2] ) # after
```

- Datapath

256B, 16B cache lines, directed mapped
```
      assign cachereq_addr_byte_offset = cachereq_addr[1:0];
      assign cachereq_addr_word_offset = cachereq_addr[3:2];
      assign cachereq_addr_index       = cachereq_addr[7:4]; // 16 sets = 4 bits
      assign cachereq_addr_tag         = cachereq_addr[31:8];
```

- Control Unit

implement FSM
```
  // Set outputs using a control signal "table"
  always @(*) begin
                              cs( 0,   0,    0,    0,    0,    0,    0,    0,    0     );
    case ( state_reg )
      //                         cache cache cache tag   tag   data  data  valid valid
      //                         req   resp  req   array array array array bit   write
      //                         rdy   val   en    wen   ren   wen   ren   in    en
      STATE_IDLE:             cs( 1,   0,    1,    0,    0,    0,    0,    0,    0     );
      STATE_TAG_CHECK:        cs( 1,   0,    1,    0,    0,    0,    0,    0,    0     );   
      STATE_INIT_DATA:        cs( 1,   0,    1,    0,    0,    0,    0,    0,    0     );     

      default:                cs( 0,   0,    0,    0,    0,    0,    0,    0,    0     );

    endcase
  end
  
```

# DIS SEC09 - Queue

Testing info can be found [here](https://github.com/cornell-ece4750/ece4750-sec09-mem-rtest-queues/blob/main/docs/index.md)
Key point is queue/memory should be initialized before the testing - by putting data first!
- in random test, data_1KB function intializes before the "read" is called in the tests

- Normal queue: without combinational 
    - enqueue(1 cycle), dequeue(1 cycle)
    - half of the time, the queue is empty
- Two-element Normal queue:
    - enqueue(1 cycle): packet can be in any one of two elements, dequeue(1 cycle)
    - by having two elements, the queue is never empty
    - but more area, edge
- Pipe queue:
    - combinational communication between en+dequeue in single cycle with one wire (deq_rdy->ene_rdy)
- Bypass queue
    - flow, enqueue when it's not ready
