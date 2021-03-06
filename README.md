# clash-bfcpu

This is an exercise project about FPGA programming. We use the functional hardware description language [CλaSH](http://www.clash-lang.org/) to design a Brainfuck CPU. A hardware Brainfuck interpreter is not a new idea, it has been done before with different approaches:

1. [brainfuck-cpu-fpga](https://gergo.erdi.hu/blog/2013-01-19-a_brainfuck_cpu_in_fpga/) (Kansas Lava)
1. [BF_CPU](http://nbviewer.jupyter.org/github/sandbender/BF_CPU/blob/master/BF_MYHDL_CPU_v2.ipynb) (MyHDL)
1. [bfcpu](http://www.clifford.at/bfcpu/bfcpu.html) (VHDL)
1. [brainfuckcpu](https://opencores.org/project,brainfuckcpu) (Verilog)
1. [The BrainFuck Machine](http://grapsus.net/74/) (7400 logic gates)

... etc.

Our design has some new features:

1. The CPU includes a programming mode in which programs can be loaded directly through UART (more on this later)
1. A number of optimizations are exploited: we start with a very naïve implementation, then apply different optimizations (work in progress)
1. When running a program, the number of clock cycles is counted and displayed on the seven-segment display (we are using Nexys 4 from Digilent, but other boards with 8 seven-segment displays certainly can be used). This makes it easy to evaluate different optimizations.

<p align="center">
  <img src="https://github.com/aufheben/clash-bfcpu/raw/master/doc/video.gif" />
</p>

## Setup & Workflow

1. Clone the repository
2. Install [stack](https://docs.haskellstack.org/en/stable/README/#how-to-install)
3. Install clash

        apt-get install libncurses5-dev
        stack setup --resolver=lts-6.24
        stack install clash-ghc
Note that lts-6.x (ghc-7.10.3) is required.
4. Generate Verilog

        ./runclash.sh
        CLaSH.Prelude> :l BF/CPU.hs
        CLaSH.Prelude> :verilog
5. Prepare top module for synthesis

        ./bf_cpu.sh

Next, we use Xilinx Vivado to synthesize the code and program the FPGA: simply create a new project, add the generated .v files as sources and Nexys4_Master.xdc as the constraint file.

It is important to configure the serial terminal properly to run programs. Take Minicom as an example:

![Minicom Config Figure 1](https://github.com/aufheben/clash-bfcpu/raw/master/doc/minicom-1.png)

Change "Serial Device" to the actual device on your computer.

![Minicom Config Figure 2](https://github.com/aufheben/clash-bfcpu/raw/master/doc/minicom-2.png)

Here I enabled "Local echo" and "Line Wrap". "Hex Display" is sometimes useful for debugging. The last "Add carriage return" doesn't work for me (or it doesn't do what I expect), the one that works is shown below:

![Minicom Config Figure 3](https://github.com/aufheben/clash-bfcpu/raw/master/doc/minicom-3.png)

Now we can run some test programs (can be found under b/). Simply type/paste the program into the serial terminal and press Ctrl+D, which will send EOT (ASCII 0x04) to the CPU. The CPU then starts executing the instructions and when the program terminates, you can input a new program. Press the "CPU RESET" button on the board to abort the execution of any program.

mandelbrot.b:

![mandelbrot.b](https://github.com/aufheben/clash-bfcpu/raw/master/doc/mandelbrot.png)

Self-interpreting is also possible: dbfi.b is world's shortest Brainfuck interpreter in Brainfuck:

![dbfi.b](https://github.com/aufheben/clash-bfcpu/raw/master/doc/self.png)

The following sections assume you already know the Brainfuck instructions.

## Naïve Implementation

The first version simply implements the operational semantics of the 8 instructions. Specificially, the `[` and `]` instructions will search through the instruction memory to find a matching bracket, which is very inefficient.

But this version is also the "non-cheating" version since we treat the Brainfuck instructions **as is**, no transformation to the original program is performed. _So in a sense this version is the true Brainfuck CPU :-)_

Note that the input is not buffered (there is no FIFO after Rx). Although this is sometimes not desirable, we find it easier to work with in some interactive cases.

## Optimizations

The main reference of optimization is [brainfuck optimization strategies](http://calmerthanyouare.org/2015/01/07/optimizing-brainfuck.html).

The first bottleneck is IO: UART is quite slow. At 19200 baud rate, it takes 52320 clock cycles to transmit 1 byte. If there is no buffering at Tx, hello.b takes 680487 cycles to run, while mandelbrot.b takes 32217824107 cycles. Since Nexys 4 runs at 100MHz, mandelbrot.b takes about 322.5s! When a 2^8 bytes FIFO is added to Tx, hello.b takes _ cycles and mandelbrot.b takes _ cycles. Quite an improvement! Next, it is natural to turn up the baud rate.

We take 3 programs to benchmark our optimizations: hanoi.b, long.b, mandelbrot.b. The other 3 programs in the reference post are not used as they require inputs, which makes it hard to measure the precise running time unless we add a stage to allow the user to supply the input before the program runs.
