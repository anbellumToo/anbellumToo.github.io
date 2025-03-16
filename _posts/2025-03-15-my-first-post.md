---
layout: single
title: "Clock Domain Crossing"
date: 2025-03-15
categories: concepts
tags: [jekyll, minimal-mistakes]
author_profile: true
---
[What is Clock Domain Crossing?](#what-is-clock-domain-crossing)

[Solutions](#clock-domain-crossing-solutions)
- [Flip-Flop Synchronizers](#flip-flop-synchronizers)
- [Binary and Gray Encoding](#binary-and-gray-encoding)

[FIFO Example](#fifo-example)

## What is Clock Domain Crossing?

Clock Domain Crossing (CDC) occurs when data transfers between different clock domains that may differ in frequency or phase. In a multi-clock system, data is first updated synchronously with the edge of the source clock, then it must be safely captured and updated synchronously in the destination clock domain. While simple in theory, there are challenging aspects to consider when attempting to maintain the reliability of data transfer. Signals that originate from one clock domain may not meet the setup and hold time required by the destination clock. Incorrect handling of CDC will lead to timing violations, metastability, data corruption, and other unpredictable behaviors in digital systems. 

Though timing issues are a concern when implementing CDC, there are many performance benefits to using multiple clock domains in a digital design. 
- Decrease the clock rate for some parts of the design while using a higher clock rate when necessary in order to improve power efficiency.
- Data throughput can be improved for a high-speed communication protocol while internal interactions operate at a lower clock rate.
- There is more design flexibility when utilizing multiple clocks in a single design. 
- Improves clock tree synthesis results for skew and jitter when there are multiple localized clocks rather than one global clock. 

## Clock Domain Crossing Solutions

To achieve reliable data transfer, various CDC techniques are used. Some strategies include flip-flop synchronizers, handshake protocols, and FIFO-based solutions. FIFO buffers are particularly useful when handling bulk data transfers across clock domains, which we will explore in detail.

#### Flip-Flop Synchronizers

Among the most simple ways to solve CDC timing issues is through a flip-flop synchronizer. This uses (usually) two sequential flip-flops triggered by the destination clock. The flip-flop synchronizer solution will allow enough time for a signal to settle and avoid metastability.

```Systemverilog
module ff_synchronizer (
    input  [0:0] clk_dest_i,      // destination clock 
    input  [0:0] rst_dest_i,      // destination reset
    input  [0:0] async_signal_i,  // async input signal
    output [0:0] sync_signal_o    // sync output signal
);
    logic [0:0] stage1_l;  // first stage 
    logic [0:0] stage2_l;  // second stage 

    // two sequential flip-flops
    always_ff @(posedge clk_dest) begin
        if (rst_dest_i) begin
            stage1_l <= 1'b0;
            stage2_l <= 1'b0;           
        end
        else begin
            stage1_l <= async_signal_i;
            stage2_l <= stage1_l;
        end
    end

    assign sync_signal_o = stage2_l;

endmodule
```

#### Binary and Gray Encoding

Transferring a data bus is more challenging as binary encoding is unsafe due to the possibility of multiple bits changing in one increment or decrement. This may cause data corruption from some bits updating on the first destination clock synchronizer while other bits remain in their previous state. This is a massive problem for digital design, for example, think about a state machine encoding. If state changes and is passed to the destination clock, corrupted bits may cause the state encoding to point to a nonexistent state. 


Instead, Gray code is used because only one bit changes per increment or decrement, making it safer for synchronization as it will either stay in the form of the previous data or update to the correct current data.

```Systemverilog
module bin_to_gray
  #(parameter width_p = 5)
  (input [width_p - 1 : 0] bin_i
  ,output [width_p - 1 : 0] gray_o);

   genvar i;
    generate
      for(i = width_p -1; i > 0; i = i - 1) begin
        assign gray_o[i - 1] = bin_i[i] ^ bin_i[i - 1];
      end
    endgenerate

    assign gray_o[width_p-1] = bin_i[width_p-1];

endmodule
```

```Systemverilog
module gray_to_bin
  #(parameter width_p = 5)
   (input [width_p - 1 : 0] gray_i
    ,output [width_p - 1 : 0] bin_o);

  genvar i;
  generate
    for(i = 0; i < width_p; i++) begin
      assign bin_o[i] = ^(gray_i >> i);
    end
  endgenerate

endmodule
```


## FIFO Example
