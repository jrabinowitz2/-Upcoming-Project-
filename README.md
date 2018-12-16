# MINESWEEPER: REVERSE & EXPLOIT CHALLENGE

As an exercise in reverse engineering I decided to reverse the version of Minesweeper that comes with Windows XP. My goal was to learn enough about the game to build some sort of cheat code or 'trainer' program (manipulate the timer, infinite flags, etc.). However, when I was finished reversing the relevant code chunks I came across a post online where someone suggested a challenge: try to modify the binary so that the game always starts with the mines already flagged. The post also claimed there is an elegant solution to this problem which only requires modifying **ONE LINE OF CODE**.

![reversing_process](/images/reversing_process.png)

Reversing the game wasn't too difficult, but completing the aforementioned challenge took some patience and careful thought. This challenge was great practice and I'd highly recommend trying to solve it, but if you're stuck follow along with the walkthrough below. Enjoy!

## REVERSE ENGINEERING

### STATIC ANALYSIS

Before beginning dynamic analysis, I checked the file headers using PEview to determine whether or not Minesweeper XP is a relocatable module with ASLR enabled. Address Stack Layout Randomization (ASLR) is a memory-protection used by some binaries to defend against buffer overflow attacks. It makes exploitation more difficult by randomizing the location in memory where the executable is loaded each time.

![static_analysis](/images/static_analysis.png)

An ASLR-enabled module will have an optional header with the *IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE (x0040)* flag set in the DllCharacteristics field. Looking at the value in PEview, I saw that Minesweeper XP was compiled without ASLR; this would make building a cheat code or trainer easy, as I could hardcode addresses.

### DRAW_TILES ROUTINE

My first move in the debugger was to look at the names of the import functions and find one that might be used to draw the board. I quickly noticed the GDI32 library, which is used to interact with graphics device drivers. Within the GDI32 functions one called *"BitBlt"* looked promising. A quick look at the Microsoft docs yields this description:

>"The BitBlt function performs a bit-block transfer of the color data corresponding to a rectangle of pixels from the specified source device context into a destination device context."

Perfect. I found where this function was referenced, and noticed it was within a nested loop. Now it was obvious this part of the code handles drawing the game tiles; looking at the BitBlt function I could see the game height and width on the stack, and tracing these variables further back in the code I could see they came from memory locations x1005338 and x1005334, respectively. I confirmed these values, along with the number of mines/flags (which was nearby), by customizing them and observing the changes in memory.

![draw_tiles_1](/images/draw_tiles_1.png)
![draw_tiles_2](/images/draw_tiles_2.png)

### DRAW_FLAG ROUTINE

Having found the DRAW_TILES routine, my plan was to then find the DRAW_FLAG routine, and find a way to call the latter from within the former (I should note here, the routine I call DRAW_FLAG actually handles all tile clicks, not just flags). I found another reference to BitBlt in a short routine that has familiar operations involving rows and columns.

![draw_flag](/images/draw_flag.png)

The instruction at x1002669 loads from an address created by [row+col+1005340]. This was a hint that the minefield is an array in memory starting at x1005340. By looking at the minefield in various states I deduced the different codes used to represent the different states of the tiles.

![notepad](/images/notepad.png)

### MINEFIELD INITIALIZATION

Now that I understood the DRAW_FLAG routine I realized that solving the challenge involved manipulating the minefield itself, rather than the drawing routine. I needed to find the routine that initialzed the minefield in memory. Intuitively, the routine must use a 'random' function in order to randomize the minefield with each new game. To find it I looked back at the import list and found the rand() function, then where it is referenced.

![init](/images/init.png)

Clearly I had found the initialization routine. It replaces the current width/height values with the new ones, then uses them to build the empty minefield. After this, the random function is called twice- once for the row and once for the column- to select a random tile. The code then checks if there's already a mine in that location. If so, another random tile is selected. If not, a mine is placed there. This loop ends when all mines have been placed.

## COMPLETING THE CHALLENGE

With the reverse engineering out of the way, I was ready to approach the challenge problem. It was now clear I had to modify the minefield initialization routine to implement the following (in pseudocode):

    if tile has mine (tile==x8F):
        flag (tile=x8E)
    else (tile==x0F):
        no flag (tile=x0F)

I thought about translating this to assembly and placing in a null space in the program's .text section. Then I realized that would definitely translate to more than one line of assembly instructions, and I remembered the post that claimed there exists an elegant, one-line hack for this problem.

After at least another 20-30 minutes of brainstorming, I became sure I needed to modify an instruction rather than add my own. I then found the pertinent line and made the following modification (shown in RED):

![modified_init](/images/modified_init.png)

To test it out I removed all breakpoints and reset the program with my one modification. Then I hit run and saw this:

![success](/images/success.png)

:boom:It worked!:boom: 
Just to be sure, I hit the smiley several more times to watch the flag configurations randomize. I then enjoyed a very quick and easy game of Minesweeper XP.

## THOUGHTS

Overall, this was a great learning experience- both practical and fun. This sort of exercise is an effective way to practice skills that apply to malware analysis as well as forensics.
