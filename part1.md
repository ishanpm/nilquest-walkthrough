# Nil Quest

Nil Quest is an unusual game by PICO-8 forum user "thisismypassword". The premise is that you must defeat the dragon near your village, a task common to many adventure games. This is made especially difficult by the fact that you have no sense of sight, hearing, touch, or pain. Completing this challenge will require some unusual methods, and a surprising amount of knowledge about computer architecture.

Before you start reading, though, I highly encourage you to follow along if you own a copy of PICO-8. This is definitely not the only way to find the solution, it is merely one possible path. All of the files I created while solving this are included, but I recommend that you try creating your own versions if you can. Those files have spoilers, so don't read through them too soon.

## Part 1: Blind exploration

The game presents itself as a completely black screen. None of the six keys seem to do anything, and the game never produces any images or sounds. The pause menu offers no clues either. With nowhere else to look, It seems like we'll have to peek behind the scenes.

### First look

The source code is a little unusual. Line 3 defines `d`, a huge list containing 5906 numbers. Lines 5-29 define `t`, a list of short functions. The rest of the code defines functions `g`, `h`, `n`, and `l`, some variables `a`, `b`, `p`, and `s`, and finally what appears to be a short main loop.

One thing to note is that many common PICO-8 system calls are absent. This code has no way to read any cartridge data, so the sprite, map, and sound data go unused. There's nothing to read anyway, since the cart data seems to just be filled with the numbers `78 73 76 33` over and over (which spell "NIL!" when converted to ASCII). There are also no commands to draw to the screen or produce sounds at all, aside from a `CLS()` at the beginning. There isn't even an `_UPDATE` function defined. However, the functions `FLIP()`, `BTN()`, `STAT()`, and `LOAD()` are present inside the list of functions `t`.

`FLIP()` allows the game to manually go to the next frame, without the need for an `_UPDATE` loop. `BTN()` allows the game to determine what buttons are currently pressed. `STAT()` allows reading many PICO-8 system variables, such as CPU usage or time of day. `LOAD()` allows starting another cartridge from the filesystem or PICO-8's online BBS. Even without any traditional output, these functions still allow the game to maintain some connection to the player.

### Prettifying

What exactly is this program doing? To understand it, we have to take a look at the single-letter-named functions and work out what they're for.

The first one is named `g`. It appears to simply return the `p`th number from `d`, then increment `p`. In this way, repeated calls to `g` will advance through `d`, one number at a time.

The next function, `h`, calls `g` twice, meaning that it reads off two numbers from `d`. It adds the two numbers, shifting the second left by 8 bits. This suggests that the numbers in `d` are meant to be treated as bytes; indeed, each one is an unsigned byte between 0 and 255.

Next is `n`. This function calls `h`, reading off the next two numbers from `d`. It then checks if the variable `q` is odd or even. If odd, then the value from `h` is used as an index for `d`; if even, the value is just returned as is.

Next, `l`. This function is different from the other ones, since it takes an input. It appears to scan through `d`, converting each number to a character in a string. It produces two strings in this manner, each terminated by a byte set to 255. This function is only used in one place, to produce input for `LOAD`. The first string corresponds to a cartridge name, and the second string corresponds to the input data given to that cart.

The main loop should show how these functions all come together. It consists of just 3 lines. First, a number is read from `d`, and assigned to `q`. The number is then used to choose a function from `t`. The number is divided by 2 first, so each function can be referred to with an odd number or an even number. This odd/even information can later be used in the `n` function. Finally, the chosen function is called.

It seems that the true purpose of the numbers in `d` has been revealed: Each one indicates an instruction in `t` to call. As the counter `p` increases, the execution of the mysterious payload advances. Most of the functions in `t` also call either `h` or `n`; this uses the next two numbers as part of the instruction as well, making most of the instructions require a total of 3 numbers. In essence, this game is actually an emulator for a virtual microprocessor, which is where the real game runs! Ain't that something?

### Disassembly

Since nothing is being drawn to the screen, we are free to use it to display debug information. I will use it to display every instruction that is executed, so the path the execution takes through `d` can be seen. I made a table with mnemonics for each instruction in `t`, as well as whether or not each instruction reads additional values from `d`. I then added to the main loop, making it print the instructions as they ran, along with the current `p` value. I also added a loop to call `FLIP` repeatedly in the main loop, to slow the program down. Now, some semblance of progress can finally be seen as the adventurer explores the land.

The program quickly enters a loop from `p` = 1726 to 1787. This appears to be an input loop, since it calls BTN and FLIP repeatedly. Pressing a button makes it exit the loop briefly, to handle the input. Certain actions result in the program locking up in a short endless loop from 1939 to 1940, perhaps because the character has died.

Seeing the assembly of the program as it runs is useful, but it would be more useful to have the entire program written out. I wrote a script in Python to convert the list of numbers to their corresponding mnemonics, so the inner workings of the hidden program can be inspected.

Here is the table of instructions, and the mnemonics I used. Note that every instruction can be called with two opcodes. Since most of the instructions use `n` to get their input, the input will be treated as a literal number if the opcode is even, while it will be treated as an address in `d` if the opcode is odd.

|Opcodes|Mnemonic|Length|
|-------|--------|------|
| 0,  1 |a <-    |3     |
| 2,  3 |a ->    |3     |
| 4,  5 |[a] ->  |3     |
| 6,  7 |[a] <-  |3     |
| 8,  9 |add     |3     |
|10, 11 |sub     |3     |
|12, 13 |eqa     |3     |
|14, 15 |lta     |3     |
|16, 17 |bit     |3     |
|18, 19 |and     |3     |
|20, 21 |or      |3     |
|22, 23 |xor     |3     |
|24, 25 |jmp     |3     |
|26, 27 |jtr     |3     |
|28, 29 |jfa     |3     |
|30, 31 |call    |3     |
|32, 33 |ret     |1     |
|34, 35 |push    |1     |
|36, 37 |pop     |1     |
|38, 39 |flip    |1     |
|40, 41 |btn     |1     |
|42, 43 |stat    |1     |
|44, 45 |ldcart  |1     |

### Invalid operation

Oddly, one instruction seems to be called frequently, but it has no implementation. This instruction, 94, would be `t[48]` if it existed. `a` seems to be set to specific values before this instruction, only to be rewritten shortly afterwards. Perhaps this instruction is meant to somehow communicate information back to the player.

Modifying the debug display to print the value of `a` whenever instruction 94 is called shows that calls to this mystery instruction correspond with actions the player takes. Exploring with the arrow keys seems to suggest that the adventurer is moving along a square grid, with each square identified by particular values given to instruction 94. Other events are also identified in this manner. For example, the initial room shows 1943, hitting a wall prints 5413, and attempting to interact yields 5435. No matter what you do, the messages 5724 and 5746 appear, apparently foreshadowing an inevitable death that happens after 43 inputs.

In addition to the mysterious unused opcode, there is a large portion of the decoded program that doesn't seem to do anything useful. It is filled with many redundant assignments and useless operations that don't look like real code. This region seems to begin just after the death loop, at address 1943.

### Hidden messages

Hold on - 1943? That number is familiar. Perhaps the input provided to the unused instruction is really a pointer to this area. That would mean that this region is not code, but a different type of data.

A good guess for the type of this unknown data would be text. The number 255 appears several times, which is used inside of `l` as a string terminator. However, simply using `l` directly yields garbage. The list of characters it uses to convert doesn't even have the full alphabet. Maybe a different character encoding would work.

Some experimentation reveals that this data is, in fact, text! The alphabet it uses is this: ` abcdefghijklmnopqrstuvwxyz???'-` The three `?` characters are never actually used, and the `-` is a statement separator, used to break up a message into small chunks. Messages are separated by the number 255, similar to the `l` function. Applying this alphabet to the list of numbers reveals that most of the information stored in `d` is not code, but readable text. The resulting text dump contains snippets that describe an adventurer collecting and using items, fighting a dragon, and even an interesting tidbit about a secret room. This must be what the game is meant to display.

We can now write an implementation for the missing instruction `t[48]`. I'll copy the definition of `l` into a new function with this alternate alphabet, which I'll call `decode`. This function requires some additional modification to cut off lines at 32 characters, to fit on the PICO-8 screen, but it will then be a quite suitable definition for the missing instruction. With this, descriptions of the character's actions are printed to the screen, finally offering real feedback for the input.

Now that we have access to descriptions of everything, we can finally play the adventure game like an actual game! We're not finished with the source code, though - this adventure is anything but straightforward.

[Part 2: The quest](part2.md)