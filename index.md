---
layout: home
tags: fpga vga verilog
categories: Project
---

<h1>Waving South African Flag <img src="https://cdn.jsdelivr.net/gh/twitter/twemoji@latest/assets/72x72/1f1ff-1f1e6.png" height="40" alt="South African flag"></h1>

I used a Basys 3 FPGA board and Verilog to draw the South African flag on a VGA monitor, then took it further by making the flag wave at the press of a button and writing "SOUTH AFRICA" across the green band. This post covers the template I started from, how I adapted it, and the problems I ran into along the way.

## **Template VGA Design**
### **Project Set-Up**
This project was created in Vivado as an RTL project trageting the Basys 3 board. The design flow was:
- Add the verilog sources (VGATop, VGASync and VGASouthAfrica)
- Add the Basys 3 constraint file
- Generate a CLocking wizard IP (clk_wiz_2) to produce a 25 MHz pixel clock from the board's 100 MHz oscillator
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
The official design specifies a 2:3 flag where the green Y band is one fifth of the height and the white and gold borders are one fifteenth each, with the arms of the Y running from the corners to the centre. Once the static flag worked, I added two extras: a waving animation while the centre button (btnC) is held, and the words "SOUTH AFRICA" in the green band. 

### **Code Adaptation**
I replaced the template's colour module with my own module, VGASouthAfrica. The flag is symmetric about its middle row, so I first calculate d, the distance from the middle row, which lets me describe only the top half. On the 640x480 screen the centre line of each Y arm runs from (0, 0) to (320, 240), a slope of 4/3, so the value X = 3*col + 4*d stays constant along every diagonal edge. 
Each coloured band is then simply a range of X (for example the black triangle is X < 575), and the horizontal bar of the Y is a range of d. The colour of each pixel is chosen by drawing the regions in order, with later regions overwriting earlier ones: background, red, blue, white, green, text, gold, then black. I also tuned the 4-bit colours to be closer to the official shades (for example 12'h074 for green and 12'hD43 for red) rather than pure primary colours.
```
wire [12:0] X = 3*col + 4*d;
wire in_black = (X < 575);
wire in_gold  = (X >= 575) && (X < 729);
wire in_green = ((X >= 729) && (X < 1191)) || ((d < 48) && (X >= 729));
wire in_white = ((X >= 1191) && (X < 1345)) || ((d < 80) && (X >= 729));

if (row_w < 240) {red_next, green_next, blue_next} = 12'hD43;
    else             {red_next, green_next, blue_next} = 12'h018;
    if (in_white) {red_next, green_next, blue_next} = 12'hFFF;
    if (in_green) {red_next, green_next, blue_next} = 12'h074;
    if (in_text)  {red_next, green_next, blue_next} = 12'hFFF;
    if (in_gold)  {red_next, green_next, blue_next} = 12'hFB2;
    if (in_black) {red_next, green_next, blue_next} = 12'h000;
```

### **Simulation**
Show how you simulated your own design. Are there any things to note? Demonstrate your understanding. Add a screenshot. Guideline: 1-2 short paragraphs.

### **Synthesis**
Describe the synthesis & implementation outputs for your design, are there any differences to that of the original design? Guideline 1-2 short paragraphs.

### **Demonstration**
The flag displays correctly on the monitor, with the text in the green band, and it waves while the centre button is held.

<video controls width="640" height="320">
  <source src="https://lowrands.github.io/SOC-Project-vertilog/docs/assets/images/Project%20video.mp4" type="video/mp4">
</video>


