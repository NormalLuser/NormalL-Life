# NormalL-Life
NormalLuser’s very normal version of Conway’s Game of Life for 6502 and the Worlds Worst Videocard


My test pattern has a Glider Factory, a Light-Weight Spaceship, a Pulsar, a Penta-decatholon among others and runs at at a glorious 2.5 FPS. 
Done with only 1.4 Mhz of 8 bit CPU.  

This otherwise straightforward version of Conway’s Game of Life relies on a unique peculiarity of the hardware on the Worlds Worst Video Card to allow an elegant single pass algorithm on the 6502 CPU.

Conway’s game of life has simple rules, but on limited systems there is one thing that gets in the way.
You need to keep the current data and still update the screen for the next generation.
	
The Ben Eater 6502 and VGA kits give you a 6502 system with 16 kb of ram, but the top 8 kb is the screen area. It is a very simple screen buffer. There is no hardware to change where the screen area is in ram or anything like that. If you put something in a byte in the screen area it shows up on the screen as a pixel instantly. If you want to change it you have to draw all the pixels yourself in RAM. Even if we worked around Zero-Page and clobbered the Stack to use most of our RAM as a buffer we would still need to spend many cycles copying data to the screen buffer. Too slow!

That VGA hardware is 100 pixels by 64 pixels when attached to the 6502 like in the Ben Eater Video series.  It is 64 colors per pixel. This means that 2 bits are used each for Red, Green, and Blue. This 6 bit color leaves 2 bits left over of the 8 bits for every pixel. These top two bits are not hooked up to any hardware at all. 
In other words; There are 8 bits dedicated to each pixel in RAM but only 6 bits change the color. 
This can be used to great advantage!

My code has a simple single pass implementation made possible by this unique video hardware.
On each pass I move across the line starting at the top and count the neighbor pixels that are white.

I process the game of life rules to decide if the pixel needs to be changed to either on or off. When I change the pixel to black or white I also I set the top unused bit. This will not change how the pixel looks on the screen, but it does tell us the value of that cell prior to the change. 

When counting cells I look at that top bit on the pixels above and behind of my target pixel, and on the pixels ahead and below my target I drop the top bit and look at the current value of pixel.

Since the unused bit I use to mark the pixel is the top bit I can use the BPL ( Branch on Result Plus ) opcode. The nice thing about this opcode is that while the name suggests that you only use it while in decimal mode and working with positive and negative numbers, you don’t have to only use it then.
On the 6502 when the top bit is set, even if you are not in decimal mode, it will change the Negative flag and allow you to branch on it.

So I can simply do this:

     LDA (Neighbor1),y  ; Load Pixel above target 
	 BPL SkipINX      ; Skip INX if top bit not set
	 INX              ; Bit must be set, count it
	SkipINX:          
       

   

This counts the pixel if the prior state was filled, while ignoring the current state.


To count a pixel’s current state I just do this:

 
     LDA (Neighbor7),y  ; Load Pixel below target 
	 AND #%00111111   ; Remove top bits to see current value
	 BEQ SkipINX      ; Skip if Zero
	 INX              ; Must be filled, count it
	SkipINX:
	
 


I use the neighbor count from code like this to do the the game logic.
The first thing I do after the count is load the target cell and drop the top bits to get the current value. 
I then do BNE to branch on the cell being zero or non zero.

Since our fist step is deciding if the target is ‘alive’ or not our logic from there allows us to assume the prior state of the pixel when updating the current state.
Meaning:

When we are marking a pixel ‘alive’ we already know that it was ‘dead’ and can always make the top bit 0 by storing $3F. 

If we are marking a pixel ‘dead’ we already know that it was ‘alive’ and we can make the top bit 0 by storing $80.

If we are keeping a pixel ‘alive’ we know that it was alive before and can always store $BF to mark the top bit 1.

If we are keeping an empty cell empty we know that we can always store 0 to mark the top bit as previously empty and the lower bits as currently empty.


This makes for a tidy little routine that is rather snappy for a naive algorithm.

The ROM based program is just 264 bytes and uses just 18 bytes of zero-page.
This program runs at 95 cycles per cell and 2.3 generations a second. The speed does not change more than a few hundred cycles regardless of the number of changed pixels since every pixel is always checked.

A simple optimization is to use self-modifying code for some pointers.

It turns out that the pointers for the neighbor cell count are only used in one place. There is a little overhead to change the row update routine to update the memory location of the Absolute Indexed Y address in the code vrs updating a Zero-Page page location for the Indirect Indexed y. 

However the cycles saved by doing that means that for each pixel in the line we are saving cycles.

Over 47,000 cycles are saved this way getting us to 2.5 generations a second at 87 cycles a pixel for the self modifying RAM based version at the cost of 43 more program bytes.

I have a few more things to speed it up planned and I also want to make it ‘wrap around’ so pixels going off one edge come in the opposite side. 

I wonder how fast I can get this?
