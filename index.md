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
Describe the synthesis and implementation processes. Consider including 1/2 useful screenshot(s). Guideline: 1/2 short paragraphs.

### **Demonstration**
Perhaps add a picture of your demo. Guideline: 1/2 sentences.

## **My VGA Design Edit**
For my own design I chose the flag of South Africa. It is a good challenge because, unlike stripes, it has diagonal edges, a triangle, and bands of different thicknesses that all have to line up. 
The official design specifies a 2:3 flag where the green Y band is one fifth of the height and the white and gold borders are one fifteenth each, with the arms of the Y running from the corners to the centre. Once the static flag worked, I added two extras: a waving animation while the centre button (btnC) is held, and the words "SOUTH AFRICA" in the green band. 

### **Code Adaptation**
Briefly show how you changed the template code to display a different image. Demonstrate your understanding. Guideline: 1-2 short paragraphs.

### **Simulation**
Show how you simulated your own design. Are there any things to note? Demonstrate your understanding. Add a screenshot. Guideline: 1-2 short paragraphs.

### **Synthesis**
Describe the synthesis & implementation outputs for your design, are there any differences to that of the original design? Guideline 1-2 short paragraphs.

### **Demonstration**
The flag displays correctly on the monitor, with the text in the green band, and it waves while the centre button is held.
<video controls width="640" height="260">
  <source src="https://lowrands.github.io/SOC-Project-vertilog/docs/assets/images/Project%20video.mp4" type="video/mp4">
</video>

<img src="https://raw.githubusercontent.com/melgineer/fpga-vga-verilog/main/docs/assets/images/VGAPrjSrcs.png">


