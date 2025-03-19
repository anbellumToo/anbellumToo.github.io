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
- [Pulse Stretching](#pulse-stretching)
- [FIFO Example](#fifo-example)

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

```verilog
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

```verilog
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

```verilog
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
## Pulse Stretching

When transferring signals from a fast clock domain to a slow clock domain, the primary challenge is ensuring that the signal remains active long enough for the slow clock to reliably sample it. Without proper handling, a short-lived pulse in the fast domain might be missed entirely in the slow domain. This section explains how to address this issue using pulse stretching and provides practical guidelines for implementation.

Consider a scenario where:

- Fast Clock (CLK_FAST): 100 MHz (10 ns period).
- Slow Clock (CLK_SLOW): 25 MHz (40 ns period).

If a pulse in the fast domain lasts only 1 clock cycle (10 ns), it might not align with the rising edge of the slow clock, or it might violate setup/hold requirements. As a result, the slow domain could miss the pulse entirely.

To ensure the slow domain reliably detects the pulse, the signal must be stretched in the fast domain. The goal is to make the pulse active for at least 2 clock cycles of the slow domain (80 ns in this example).

```verilog
module fast_to_slow_sync (
    input  clk_fast,
    input  rst_fast,
    input  clk_slow,
    input  rst_slow,
    input  pulse_fast_i,
    output pulse_slow_o
);
    logic [7:0] stretch_ff;
    logic stretched_pulse;
    logic [1:0] sync_ff;

    always_ff @(posedge clk_fast) begin
        if (rst_fast) stretch_ff <= 8'b00000000;
        else          stretch_ff <= {stretch_ff[6:0], pulse_fast_i};
    end

    assign stretched_pulse = |stretch_ff;

    always_ff @(posedge clk_slow) begin
        if (rst_slow) sync_ff <= 2'b00;
        else          sync_ff <= {sync_ff[0], stretched_pulse};
    end

    assign pulse_slow_o = sync_ff[1];

endmodule
```



## FIFO Example

With the previous tools, we can implement a CDC fifo that is capable of transfering bulk data without the threat of data corruption issues. In order to create the CDC FIFO, we must initialize a RAM interface that will read data synchronized to the producer clock and write data synchronized to the consumer clock.

```verilog
module ram_1r1w_sync
  #(parameter [31:0] width_p = 8
  ,parameter [31:0] depth_p = 512
  ,parameter string filename_p = "memory_init_file.bin")
  (input [0:0] pclk_i
  ,input [0:0] cclk_i
  ,input [0:0] preset_i
  ,input [0:0] creset_i

  ,input [0:0] wr_valid_i
  ,input [width_p-1:0] wr_data_i
  ,input [$clog2(depth_p) - 1 : 0] wr_addr_i

  ,input [0:0] rd_valid_i
  ,input [$clog2(depth_p) - 1 : 0] rd_addr_i
  ,output [width_p-1:0] rd_data_o);
   logic [width_p-1:0] ram [depth_p-1:0];

   initial begin
      for (int i = 0; i < depth_p; i++) begin
        ram[i] = '0;
      end
   end

   logic [width_p-1:0] rd_data_l;

   always @(posedge pclk_i) begin
      if(preset_i) begin
         rd_data_l <= '0;
      end 
      else if(rd_valid_i) begin
          rd_data_l <= ram[rd_addr_i];
      end
   end

   always @(posedge cclk_i) begin
      if(creset_i) begin
         ram[wr_addr_i] <= ram[wr_addr_i];
      end 
      else if(wr_valid_i) begin
          ram[wr_addr_i] <= wr_data_i;
      end
   end

   assign rd_data_o = rd_data_l;

endmodule
```

Now we implement the CDC FIFO by connecting the synchronizers, gray encoding, and the two clock port RAM module. We use the RAM module to transfer data between the two clock domains safely. Within the FIFO implementation, the read and write addresses must be transferred between clock domains in order to facilitate the logic for the ready valid handshakes between interfaces. 

```verilog
module fifo_1r1w_cdc
 #(parameter [31:0] width_p = 32
  ,parameter [31:0] depth_log2_p = 8
  )
   // the "c" for consumer, and "p" for producer interfaces. 
  (input [0:0] cclk_i
  ,input [0:0] creset_i
  ,input [width_p-1:0] cdata_i
  ,input [0:0] cvalid_i
  ,output [0:0] cready_o 

  ,input [0:0] pclk_i
  ,input [0:0] preset_i
  ,output [0:0] pvalid_o 
  ,output [width_p-1:0] pdata_o 
  ,input [0:0] pready_i
  );

    wire [0:0] full;
    wire [0:0] empty;
   
    wire [0:0] en_w;
    wire [0:0] en_r;
    wire [width_p-1:0] ram_o;

    logic [depth_log2_p:0] wr_add, wr_add_delay, 
    wr_add_bin2gray, wr_add_sync1, wr_add_sync2, 
    crossed_wr_add, rd_add, rd_add_delay, 
    rd_add_bin2gray, rd_add_sync1, rd_add_sync2, 
    crossed_rd_add, rd_add_next;

    assign en_w = cvalid_i && cready_o;
    assign en_r = pready_i && pvalid_o;

    assign full =(wr_add[depth_log2_p-1:0] === 
        crossed_rd_add[depth_log2_p-1:0]) && 
        (wr_add[depth_log2_p] !== 
        crossed_rd_add[depth_log2_p]);
    assign empty = (crossed_wr_add === rd_add);
    assign cready_o = ~full;
    assign pvalid_o = ~empty;

  // cross for wr_add from consumer to producer side
  // convert the write address to gray encoding
  bin2gray
  #(.width_p(depth_log2_p+1))
  bin2gray_inst_wr_add
  (.bin_i(wr_add)
  ,.gray_o(wr_add_bin2gray));

  //delay the grey encoded write address for 1 cc
  always_ff @(posedge cclk_i) begin 
    if (creset_i) begin
      wr_add_delay <= '0;
    end
    else begin
      wr_add_delay <= wr_add_bin2gray;
    end
  end

  // two stages of synchronizers
  always_ff @(posedge pclk_i) begin
    if(preset_i) begin 
      wr_add_sync1 <= '0; 
      wr_add_sync2 <= '0;
    end
    else begin 
      wr_add_sync1 <= wr_add_delay;
      wr_add_sync2 <= wr_add_sync1;
    end
  end

  // convert write address to bin encoding
  gray2bin
  #(.width_p(depth_log2_p+1))
    gray2bin_inst_wr_add
   (.gray_i(wr_add_sync2)
    ,.bin_o(crossed_wr_add));  

  // cross for rd_add from producer to consumer side
  // convert to gray encoding
  bin_to_gray
  #(.width_p(depth_log2_p+1))
  bin2gray_inst_rd_add
  (.bin_i(rd_add)
  ,.gray_o(rd_add_bin2gray));

  // two stages of synchronizers
  always_ff @(posedge cclk_i) begin
    if(creset_i) begin 
      rd_add_sync1 <= '0; 
      rd_add_sync2 <= '0;
    end
    else begin 
      rd_add_sync1 <= rd_add_bin2gray;
      rd_add_sync2 <= rd_add_sync1;
    end
  end

  // convert back to bin encoding
  gray_to_bin
  #(.width_p(depth_log2_p+1))
    gray2bin_inst_rd_add
   (.gray_i(rd_add_sync2)
    ,.bin_o(crossed_rd_add));  

  always_ff @(posedge cclk_i) begin
    if(creset_i) begin 
      wr_add <= '0;
    end
    else if (en_w) begin
      wr_add <= wr_add + 1;
    end
  end

  always_ff @(posedge pclk_i) begin
    if (preset_i) begin
      rd_add <= '0;
    end 
    else if (en_r) begin
      rd_add <= rd_add + 1;
    end
  end

  always_comb begin 
    rd_add_next = rd_add;
    if(en_r) begin
      rd_add_next = rd_add + 1;
    end
  end

  ram_1r1w_sync
  #(.width_p(width_p), 
    .depth_p(1<<depth_log2_p), 
    .filename_p()) 
    ram_1r1w_sync_inst
   (.pclk_i(pclk_i), 
   .cclk_i(cclk_i), 
    .preset_i(preset_i),
    .creset_i(creset_i),
    .wr_valid_i(en_w),
    .wr_data_i(cdata_i),
    .wr_addr_i(wr_add[depth_log2_p-1:0]),
    .rd_valid_i(1'b1),
    .rd_addr_i(rd_add_next[depth_log2_p-1:0]), 
    .rd_data_o(pdata_o));

endmodule
```

