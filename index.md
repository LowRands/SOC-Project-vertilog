---
layout: home
tags: fpga vga verilog
categories: Project
---
<style>
body { background-color: #2a6199; color: #e8f5e9}
img { max-width: 100%; }
video { max-width: 100%; height: auto; }
</style>

<h1><b>Waving South African Flag</b> <img src="https://cdn.jsdelivr.net/gh/twitter/twemoji@latest/assets/72x72/1f1ff-1f1e6.png" height="40" alt="South African flag"></h1>

I used a Basys 3 FPGA board and Verilog to draw the South African flag on a VGA monitor, then took it further by making the flag wave at the press of a button and writing "SOUTH AFRICA" across the green band. This post covers the template I started from, how I adapted it, and the problems I ran into along the way.

## **Template VGA Design**
### **Project Set-Up**
This project was created in Vivado as an RTL project trageting the Basys 3 board. The design flow was:
- Add the verilog sources (VGATop, VGASync and VGASouthAfrica)
- Add the Basys 3 constraint file
- Generate a Clocking wizard IP (clk_wiz_2) to produce a 25 MHz pixel clock from the board's 100 MHz oscillator
- Run synthesis
- Run implementation
- Generate bitstream
- Program hardware over USB

<img src="https://raw.githubusercontent.com/melgineer/fpga-vga-verilog/main/docs/assets/images/VGAPrjSum.png">
<img src="https://raw.githubusercontent.com/melgineer/fpga-vga-verilog/main/docs/assets/images/VGAPrjSrcs.png">

### **Template Code**
The template is split into three parts. 
1. VGATop is the top-level module: it instantiates the Clocking Wizard, the sync generator VGASync and a colour module, and wires them together.
2. VGASync uses two counters, hcount (0 to 799) and vcount (0 to 524), to step through every pixel position in a frame. From these it produces the hsync and vsync pulses, a vid_on signal that is high only inside the visible 640x480 area, and the current row and col.
3. The colour module takes row and col and decides the 12-bit colour (4 bits each for red, green and blue) of that pixel.

A VGA monitor has no frame memory of its own. The image is drawn one pixel at a time, left to right and top to bottom, 60 times a second. Each line is 800 pixel clocks long: 640 visible pixels, then a 16-pixel front porch, a 96-pixel sync pulse and a back porch. Each frame is 525 lines: 480 visible, then a 10-line front porch, a 2-line sync pulse and a back porch. 800 x 525 x 60 Hz gives the 25 MHz pixel clock. The colour outputs must be zero during blanking, so VGATop gates them with vid_on. This means the whole picture comes down to one question answered in logic: given this row and column, what colour should the pixel be?
```
if (hcount < HDISP && vcount < VDISP)
    vid_on <= 1;
else
    vid_on <= 0;
```

### **Simulation**
The design was simulated in Vivado's behavioural simulator by applying a clock and reset and watching the VGASync outputs. The key things to check are that hcount wraps at 800 and vcount at 525, that the active-low hsync and vsync pulses are 96 clocks and 2 lines long, and that vid_on is only high while row is below 480 and col is below 640. A full frame is 420,000 pixel clocks (about 16.8 ms), so the simulation has to run for at least that long to see a complete vsync cycle.

### **Synthesis**
Synthesis converts the Verilog into a netlist of FPGA primitives (look-up tables, flip-flops and the MMCM clocking resource used by the Clocking Wizard). 
Implementation then places those primitives on the chip and routes the connections between them, checking that every path meets the 40 ns period of the 25 MHz clock. Finally, a bitstream is generated and downloaded to the board. 
The template design is small, so it uses only a tiny fraction of the Artix-7's resources.

### **Demonstration**
The template displayed coloured stripes on the monitor at 640x480, 60 Hz.

## **My VGA Design Edit**
For my own design I chose the flag of South Africa. It is a good challenge because, unlike stripes, it has diagonal edges, a triangle, and bands of different thicknesses that all have to line up. 
The official design specifies a 2:3 flag where the green Y band is one fifth of the height and the white and gold borders are one fifteenth each, with the arms of the Y running from the corners to the centre. Once the static flag worked, I added two extras: a waving animation while the centre button (btnC) is held, and the words "SOUTH AFRICA" in the green band. I was assisted by AI throughout this project as it was a completely new section of my software and electronic engineering studies.

### **Code Adaptation**
I replaced the template's colour module with my own module, VGASouthAfrica and adapted VGATop and VGASync. The flag is symmetric about its middle row, so I first calculate d, the distance from the middle row, which lets me describe only the top half. On the 640x480 screen the centre line of each Y arm runs from (0, 0) to (320, 240), a slope of 4/3, so the value X = 3*col + 4*d stays constant along every diagonal edge. 
```
wire [12:0] X = 3*col + 4*d;
wire in_black = (X < 575);
wire in_gold  = (X >= 575) && (X < 729);
wire in_green = ((X >= 729) && (X < 1191)) || ((d < 48) && (X >= 729));
wire in_white = ((X >= 1191) && (X < 1345)) || ((d < 80) && (X >= 729));
```
Each coloured band is then simply a range of X (for example the black triangle is X < 575), and the horizontal bar of the Y is a range of d. The colour of each pixel is chosen by drawing the regions in order, with later regions overwriting earlier ones: background, red, blue, white, green, text, gold, then black. I also researched and tuned the 4-bit colours to be closer to the official shades (for example 12'h074 for green and 12'hD43 for red) rather than pure primary colours.
```
if (row_w < 240) {red_next, green_next, blue_next} = 12'hD43;
    else             {red_next, green_next, blue_next} = 12'h018;
    if (in_white) {red_next, green_next, blue_next} = 12'hFFF;
    if (in_green) {red_next, green_next, blue_next} = 12'h074;
    if (in_text)  {red_next, green_next, blue_next} = 12'hFFF;
    if (in_gold)  {red_next, green_next, blue_next} = 12'hFB2;
    if (in_black) {red_next, green_next, blue_next} = 12'h000;
```
The waving effect works by shifting the row used for each column. A counter advances a phase value roughly every 42 ms while the button is held, and a 16-entry half-sine lookup table gives a vertical offset that travels across the screen. 
The offset is multiplied by col, so the left edge stays pinned like a flagpole and the wave grows towards the right; all the flag geometry then uses the shifted row, row_w. 
The button is passed through a two-flip-flop synchroniser in VGATop because it is asynchronous to the 25 MHz clock. 
```
function [4:0] half_sine(input [3:0] i);
    case (i)
        4'd0:  half_sine = 0;   4'd1:  half_sine = 3;
        4'd2:  half_sine = 6;   4'd3:  half_sine = 8;
        4'd4:  half_sine = 11;  4'd5:  half_sine = 12;
        4'd6:  half_sine = 14;  4'd7:  half_sine = 15;
        4'd8:  half_sine = 15;  4'd9:  half_sine = 15;
        4'd10: half_sine = 14;  4'd11: half_sine = 12;
        4'd12: half_sine = 11;  4'd13: half_sine = 8;
        4'd14: half_sine = 6;   default: half_sine = 3;
    endcase
endfunction
```
The text uses a small 5x7 bitmap font for just the letters needed, scaled up four times, and because it also uses row_w it waves with the flag.
```
function [4:0] glyph(input [3:0] ch, input [2:0] r);
    case (ch)
        0: case (r) 0:glyph=5'b01111; 1:glyph=5'b10000; 2:glyph=5'b10000; 3:glyph=5'b01110;
                    4:glyph=5'b00001; 5:glyph=5'b00001; default:glyph=5'b11110; endcase //S
        1: case (r) 0:glyph=5'b01110; 6:glyph=5'b01110; default:glyph=5'b10001; endcase //O
        2: case (r) 6:glyph=5'b01110; default:glyph=5'b10001; endcase //U
        3: case (r) 0:glyph=5'b11111; default:glyph=5'b00100; endcase //T
        4: case (r) 3:glyph=5'b11111; default:glyph=5'b10001; endcase //H
        6: case (r) 0:glyph=5'b01110; 3:glyph=5'b11111; default:glyph=5'b10001; endcase //A
        7: case (r) 0:glyph=5'b11111; 3:glyph=5'b11110; default:glyph=5'b10000; endcase //F
        8: case (r) 0:glyph=5'b11110; 3:glyph=5'b11110; 4:glyph=5'b10100; 
                    5:glyph=5'b10010; default:glyph=5'b10001; endcase //R
        9: case (r) 0:glyph=5'b01110; 6:glyph=5'b01110; default:glyph=5'b00100; endcase //I
        10: case (r) 0:glyph=5'b01110; 6:glyph=5'b01110; 1:glyph=5'b10001;
                     5:glyph=5'b10001; default:glyph=5'b10000; endcase //C
        default: glyph = 5'b00000; //space                
    endcase
endfunction
```
My first attempts had bugs. Expressions like row/2 - 20 wrapped around to huge values because Verilog treats them as unsigned, which made parts of the shape disappear, so I rewrote the comparisons to use only addition. I also had to fix the drawing order, and to correct the triangle, whose slope did not match the arms.

### **Simulation**
The important thing about my design is timing. VGASync registers hsync, vsync and vid_on, so they appear one clock after the counters change. My colour module also registers its output, so the colour for a pixel also appears one clock later. Because both are delayed by exactly one cycle, the colours stay lined up with the sync signals and vid_on, and the image is not shifted.

In simulation the waving logic is hard to observe directly, because phase only changes every 2^20 clocks (about 42 ms of simulated time). To see it change in a reasonable simulation time, the counter width can temporarily be reduced, for example to tick[3:0], and then restored before synthesis.

### **Synthesis**
Compared with the template, my design uses more logic because of the many comparisons for the flag regions, the multiplication for the wave offset, the font lookup and the divide by 24 used to find which letter is being drawn. The divider is by a constant, so it is built from ordinary logic rather than a full divider. The clock, sync generator and I/O are unchanged from the template, apart from the extra button input, which needed its own line in the constraints file:
```
set_property PACKAGE_PIN U18 [get_ports btnC]
set_property IOSTANDARD LVCMOS33 [get_ports btnC]
```
Out of the many issues I ran into during this project, the two biggest problems was a module name mismatch: my top level instantiated clk_wiz_0 but the IP in the project was called clk_wiz_2, I added 2 when I was changing the clock frequency to 25 MHz, removed clk_wiz_0 and clk_wiz_1 but kept clk_wiz_2, so synthesis failed with "module not found". The second was a "Spawn failed: The parameter is incorrect" error, caused by the project being stored inside a OneDrive folder that Vivado could not write to. Moving the project to a short local path without spaces fixed it.

### **Demonstration**
The flag displays correctly on the monitor, with the text "SOUTH AFRICA" in the green band, and it waves while the centre button is held.

<img src="docs/assets/images/Attempt 1.jpeg">
One of my first attempts<br><br>
<img src="docs/assets/images/Attempt 2.jpeg">
A few attempts later. Getting the white<br><br>
<img src="docs/assets/images/Attempt 3.jpeg">
Getting green and white to stretch across the full screen<br><br>
<img src="docs/assets/images/Attempt 4.jpeg">
Getting closer to the actual shape of the flag, needed to remove the white around the gold<br><br>
<img src="docs/assets/images/Attempt 5.jpeg">
Change to the correct colours and got the shape corrected<br><br>
<img src="docs/assets/images/Text Addition.jpeg">
Added the text "SOUTH AFRICA"<br><br>
<video>
  <source src="https://lowrands.github.io/SOC-Project-vertilog/docs/assets/images/Project%20video.mp4" type="video/mp4">
</video><br><br>
Final result


