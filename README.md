# FPGA_CPU_MS-PAINT
Assembly code which can run on a RISC-V CPU. CPU was designed in system verilog for a Basys-5 FPGA board.
==============================================================================================================================================================================================================

INTRODUCTION:
	This manual covers the JK Paint software. The software allows users to create digital artwork using six color options and a cursor, paint brush and circle tool. 


OPERATING INSTRUCTIONS:
	Turn the software with no switches on the device flipped. The screen will load a 1x1 pixel cursor in the center of the display. 

CURSOR MODE:
While the far right switch remains off, the software will stay in cursor mode. The user may press the up, down, left, and right buttons to move the cursor in those respective directions. At any point, if the center button is pressed, the software resets and the cursor returns to the center of the screen. If the user switches from draw mode to cursor mode after creating an image, the cursor can freely move over the image without erasing it, making it a useful tool to travel to untouched areas of the canvas.

DRAW MODE:
If the far right switch is turned on, the software will enter draw mode and remain in this mode until the specified switch is turned off. In draw mode, the user will turn on one of the next six switches to the left of the mode switch to select a color. In order, from left to right, the colors of each switch are red, green, blue, yellow, black, and rainbow. The rainbow color works by rapidly and continuously incrementing the hex code for its color, resulting in a different color at each pixel. If no color switch is flipped, the software is effectively “erasing” by drawing white pixels. If multiple color switches are flipped, the software will use the color of the switch furthest to the right. The color which is currently selected will be displayed by filling the top bar of pixels on the screen. 
Once a color is selected, the user will again use the up, down, left, and right buttons to move the cursor but the cursor will now leave a trail of whatever color has been selected.
	At any point while in draw mode, the user may flip the second switch from the left to use the circle tool. Once this switch is on, the user can press the down button to increase the circle’s radius, the radius is capped at 3 and at this point the user must place the circle. The current radius chosen is displayed on the BASYS board’s seven segment display. When the desired radius is set, the user can turn the same circle switch off and place the circle. The drawn circle will feature a radius of whatever was selected, from 1-3, and center itself around the current cursor position. After the circle is placed, the user may continue drawing from the same cursor position.


SOFTWARE DESIGN:
INIT:
The program begins with an initializing block where the address of the switches is loaded along with the color of the background. The draw_background subroutine is then called, allowing the display to start off with an entirely white screen. Next, the x and y position registers are set to 40 and 30 respectively, this is the center of the screen. The cursor is drawn at this position and has the color which is set as a constant called CURSOR_COLOR.

STATE:
	In the state block, the seven segment is stored as 0 and the switch input is loaded. The switch input is ANDed with 1 to isolate the 0th bit which represents the right-most switch. If that switch is on, the program branches to the DRAW block. If not, the program continues on to the CURSOR block, this is why cursor mode is the default of the software. A majority of the blocks in this program jump back to STATE after executing because this allows the user to change from draw mode to cursor mode freely at any point.

CURSOR:
	In the cursor block, the button input, at address 0x11000200, is loaded. In hardware, the button input works by concatenating each button input where the 0th bit is right, the 1st is left, the 2nd is down, and the 4th is up. So the software runs through a series of branch statements checking the button input against a series of values: 1, 2, 4, and 8. For example, if the left button was pressed, the input would be 2 and the program would branch at the second branch statement, taking the program to the CURSOR_LEFT block.

CURSOR_LEFT (and right, down, and up):
	All of the directional blocks for the cursor start with a check to ensure that the cursor cannot move off screen. If the cursor’s position is on the edge of any direction, the program just jumps back to the STATE block, only allowing the cursor to move in an acceptable direction. After this check, the program sets the color register to whatever color the current pixel was before the cursor was placed and draws a dot of this color. This ensures that any drawings are not erased in cursor mode. Next, the position registers are changed according to whatever directional block the program is in; for left, -1 is added to the x register. The cursor color is loaded back to the color register. Then, the program calls read_dot which loads a register with the color of the pixel at the location specified. This is how the earlier line of this block does not erase drawings. Now, with the previous color stored, the new cursor can be drawn at this location. Finally, a delay function is called to help smooth motion and the program returns to the STATE block, awaiting the next input.

DRAW:
	In the draw block, the current color is stored in a second register to be used for a later check as it could change each time the draw block is entered. The select_color subroutine is then called to see what color switch was turned on, this may change the value of the color register if a new color was selected. A register which holds the radius of the circle is then initialized to 0 and the program checks if the switch for circle tool is on, if it is the program branches to the CIRCLE block. If not, the program checks the current color selected against the register it stored the color in before select_color. If the two colors are different, it saves the cursor position in temporaries, draws a horizontal line of pixels on top of the screen with the current color, and sets the cursor back to where it was. If the colors are the same, meaning no change in drawing color, the horizontal line doesn’t need to change and the program continues. The next section of the DRAW block functions similarly to the cursor block where it loads the button input and branches accordingly.

COLOR_SELECT:
	Is called at the beginning of the draw function and checks to see which color switches are on and modifies the a3 register since this is the register we designated to be the color of our draw. If none of the switches are on then this acts as our erase tool by setting the color to white which is also the same as the background.

DRAW_(direction):
	In the directional blocks for draw, the program again checks the cursor position in order to keep in on screen. Then, draw_dot is called to place a pixel of whatever color was selected at the cursor position. After this, the cursor’s position is moved in whatever direction specified by the button and the cursor color is loaded back to the color register. The cursor is then drawn at whatever new location was set. Like before, a delay function is called to help smooth motion and the program jumps back to STATE.

CIRCLE:
	In the circle block, the seven segment is updated to display the radius currently selected in the radius register (this starts at 1, no circle of 0 radius). The switch input is then loaded to check if the circle switch has been turned off. If it has been turned off, the program jumps to PLACE_CIRCLE. If not, it checks if the radius register is 3 because that is the max allowed radius. If it is 3, the program jumps to the line where the switch input is loaded, looping these lines of code until the circle switch is turned off to place the circle. If the radius is not 3 yet, the button input is loaded and it checks if the bottom button (increase radius button) has been pressed. If it hasn’t the program jumps to the top of the circle block, basically allowing the user as much time as desired to press the button to increase radius. When the increase radius button is pressed, the radius register is incremented by 1 and the seven segment is updated to display the new radius. Two delay functions are called to prevent the program from running too quickly and then it jumps back to the top of the circle block, to repeat the process. 

PLACE_CIRCLE:
	This block simply uses three branch statements to determine what the radius register was set to and branches to either RADIUS_1, RADIUS_2, or RADIUS_3 accordingly.

RADIUS_(radius register):
	These blocks save the cursor position then draw a circle of the specified size around the cursor by repeatedly changing the x and y registers and calling draw_dot. At the end, the x and y registers are restored to the cursor position and the program jumps back to STATE.


