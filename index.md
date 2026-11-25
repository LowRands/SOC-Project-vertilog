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
- add the verilog sources (VGATop, VGASync and VGASouthAfrica)
- add the Basys 3 constraint file
- generate a CLocking wizard IP (clk_wiz_2) to produce a 25 MHz pixel clock from the board's 100 MHz oscillator
- run synthesis
- run implementation
- generate bitstream
- program hardware over USB

<img src="https://raw.githubusercontent.com/melgineer/fpga-vga-verilog/main/docs/assets/images/VGAPrjSum.png">

### **Template Code**
Outline the structure and design of the Verilog code templates you were given. What do they do? Include reference to how a VGA interface works. Guideline: 2/3 short paragraphs, consider including screenshot(s).

### **Simulation**
Explain the simulation process. Reference any important details, include a well-selected screenshot of the simulation. Guideline: 1/2 short paragraphs.

### **Synthesis**
Describe the synthesis and implementation processes. Consider including 1/2 useful screenshot(s). Guideline: 1/2 short paragraphs.

### **Demonstration**
Perhaps add a picture of your demo. Guideline: 1/2 sentences.

## **My VGA Design Edit**
Introduce your own design idea. Consider how complex/achievabble this might be or otherwise. Reference any research you do online (use hyperlinks).

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


