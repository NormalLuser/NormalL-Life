# NormalL-Life
NormalLuser’s very normal version of Conway’s Game of Life for 6502 and the Worlds Worst Videocard

This otherwise straightforward version of Conway’s Game of Life relies on a unique peculiarity of the hardware on the Worlds Worst Video Card to allow an elegant single pass algorithm.

Conway’s game of life has simple rules, but on limited systems there is one thing that gets in the way.
You need to keep the current data while you update the next screen.
The Ben Eater 6502 and VGA kits give you a 6502 system with 16 kb of ram, but the top 8 kb is the screen area. It is a very simple screen buffer. There is no hardware to change the screen area in ram or anything like that. If you put something in a byte in the screen area it shows up on the screen, and if you want to change it you have to draw all the pixels yourself in RAM. Even if we worked around Zeropage and clobbered the stack we would still need to spend many cycles copying data to the screen buffer.

Some specifics here. That VGA hardware is 100 pixels by 64 pixels when attached to the 6502 like in the Ben Eater Video series.  It is 64 colors per pixel. This means that 2 bits are used for each Red, Green, and Blue pixel. This 6 bit color leaves 2 bits left over of the 8 bits for every pixel. These top two bits are not hooked up to any hardware at all. 
In other words; There are 8 bits dedicated to each pixel in RAM only 6 bits change the color. 
This can be used to great advantage!

My code has a simple single pass implementation made possible by this unique video hardware.
On each pass I move across the line starting at the top and count the neighbor pixels that are on.

I process the game of life rules to decide if the pixel needs to be changed to either on or off. When I change the pixel to black or white I also I set the top unused bit. This will not change how the pixel looks on the screen, but it does tell us the pixels old value prior to the change.

When counting pixels I look at that top bit on the pixels above and behind of my target pixel, and on the pixels ahead and below my target I drop the top bit and look at the current value of pixel.
Since the unused bit I use to mark the pixel is the top bit I can use the BPL ( Branch on Result Plus ) opcode. The nice thing about this opcode is that while it’s name suggests you only use it while in decimal mode and working with positive and negative numbers, you don’t have to only use it then.
On the 6502 when the top bit is set, even if you are not in decimal mode, it will change the Negative flag.

So I can simply do this:
 LDA (Screen1),y ; Above 
 BPL SkipINX ; Use prior value 
 INX              ;
SkipINX:
This counts the pixel if the prior state was filled, while ignoring the current state.

And to count a pixel’s current state I just do this:
 LDA (Screen6),y ; Below
 AND #%00111111 ; Need to remove top bits
 BEQ SkipINX ; Skip if zero, INX if not
 INX
SkipINX:

The program is 262 bytes and uses 18 bytes of zero-page and can be run directly from ROM.
Right now I’m at 95 cycles per cell and 2.3 generations a second. The speed does not change more than a few hundred cycles regardless of the number of changed pixels.
