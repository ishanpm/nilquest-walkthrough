## Part 2: The quest

Now that the true game is revealed, we can approach this like any other adventure game. Well, not really. There are still many secrets abound. While it's probably possible to find them all without looking at the source, it's a lot less tedious to look at the game and its source together. Besides, we already have this helpful disassembly laying around, so why not put it to good use?

Around this point, I created nilquest-mod.p8, which is a patched version of the game containing the messages and a simple status bar. You can use it to follow this part if you don't feel like rewriting it on your own.

### Fresh eyes

After some exploration, a map of the environment can be drawn. I included the address of the instruction responsible for printing the description of each room, since the rest of the code for each room follows this address.

```
                 Dragon
                  1334
                    |
                   Top--Cliff
                  1170  1253
                    +
                  Gate
                   979
                    :
                   Up
                   816
                    |
       Mountains-*--/
          710    
           |             ~
Forest-Crossroads-Oasis-Tent~
 573       24      107  176
           |             ~
        Village-+-House
          306      459
```

There are several rooms with special properties. The Top of the mountain contains an elevator down to the Crossroads, allowing for faster travel. The Tent will cause the entire world to rotate when exiting, meaning that whatever key you press to leave the tent will, from then on, correspond to moving west. This applies everywhere except moving from Mountains to Up, where going forward always requires pressing the right arrow key. Two exits, marked with `+`, require a key to unlock. One exit, marked with `:`, is blocked by an annoying goat that must be chased away with a weapon. Many of the rooms contain useful items which can be picked up with Z, but only one item can be held at a time.

In addition to the rooms listed above, there is a secret room with no apparent entry. The description for this room is printed at 1088, but there is no clear connection to this room anywhere in the source code.

We have to find some way to collect everything necessary to slay the dragon in the strict 42 move limit, which the descriptions reveal to be caused by a deadly scorpion sting. However, we're just exploring right now, and the move limit makes that cumbersome. Grab your Gameshark, because we can cheat the poison away! The following snippets will alter the movement script to make the poison harmless. Simply place one anywhere between the line defining `d` and the main loop. The first set will remove the poison messages entirely, while the second will just prevent it from killing you.

```
-- no poison cheat
d[1871]=24
d[1872]=122
d[1873]=7

-- harmless poison cheat
d[1937]=122
d[1938]=7
```


### Working backwards

To understand how to complete this adventure, it would probably be a good idea to look for the win condition, and work out what is needed to satisfy it. Since the game's description says we have to defeat a dragon, the dragon's lair is a good starting point when looking for the ending.

Each room is represented in the machine code by a group of instructions. The first few instructions display a description of the room and its contents, the next few show how to respond to the player's input, and the rest contain room-specific behaviors. In the case of the dragon's lair, this is slightly different. The game immediately checks whether the player is holding specific items using memory address 5912, with each item indicated by a specific value in that address. By searching for the same number, we can infer what each item is and where to get it.

The player is immediately attacked, and must be holding items 4 or 8 to survive. This item is then removed if it was 4, or replaced with 7 if it was 8. Next, the player must use item 3 or 7. If the item was 7, the memory address 5925 must not be zero as well. Completing all of this will slay the dragon, leaving the player victorious. The game will begin a subroutine of some sort, then load an external cart using the result. Since the player can't re-enter the room between the first and second phase of this fight, the only way to survive is with item 8, while address 5925 is set.

The subroutine to equip the player with an item is called using the instruction `call 1626`.  Searching for this instruction will reveal all the places where items are given to the player, as well as the messages that correspond to each item. Searching in this manner shows that item 8 indicates the player is holding both a shield and a sandwich, the sandwich being a combination of the sword and the odd stone on the cliffside. Address 5925 is set to 1 when the player drinks the attack potion in the secret room. Now, we just have to figure out how to get these items.

But maybe we can skip all that? Sneaking in `p=1497` just before the main loop should skip the entire game, running the victory routine immediately. And sure enough, it does, but the game somehow fails to load the reward. A closer look at the victory routine reveals that it is reading data starting at address 5926, which is where the input subroutine keeps a list of all the moves the player has made. With no input to read, the subroutine glitches out. Whatever cart it's loading probably verifies that the inputs we made actually complete the game, so we will have to avoid using cheats when completing the game for real. (In fact, the victory screen is encrypted with AES, the same technology used to protect data throughout the internet. We're definitely not meant to hack this part!)

### Secret room

Let's look for the attack potion first. It's in the secret room, which starts at address 1085. However, there is no reference to this address anywhere in the code. How can the player go here if there is no route?

There is one subroutine with no obvious purpose at address 1675. This increments a value at 5909 if `a` = 3, and writes some data when that value reaches 12. At the end, it jumps to 1585, the default move handler. Or so it seems...

The addresses this code writes to are not for storing variables. These actually change the code itself, creating behaviors not present in the disassembly. Similar to the cheat codes above, by rewriting the numbers after an instruction, they change where certain jumps lead. Two of the writes change the last instruction of this subroutine from `24 49 6` (`jmp 1585`) to `24 61 4` (`jmp 1085`), and the other two change the instruction at 1108 from `26 208 3` (`jtr 976`) to `26 58 2` (`jtr 570`). This changes the exit of this subroutine from the default action to the secret room, and changes the secret room's exit from the Gate to the Forest.

The Forest also has some self-rewriting mechanisms. The default action is changed from `30 49 6` (`call 1585`) to `30 139 6` (`call 1675`), changing the default action to the secret subroutine. Putting it all together, when in the Forest, moving down (`a` = 3) will do nothing the first 11 times, but on the 12th time onwards, it will instead lead to the secret room. 

After solving that polymorphic puzzle, we are rewarded with the attack potion!

### Missing key

The other condition to kill the dragon was to have the shield and the sandwich. To get the sandwich, you can shoo the goat using the axe in the Oasis, then use the key from the tent on the gate, then grab the stone and use it on the sword. To get the shield, simply use the key from the tent on the door to your house. (Why was it locked in the first place? This village isn't satisfied with sending a blind, deaf, anaptic person to their probable death, they have to steal his keys too?)

Putting aside the huge toll on our valuable 42 moves, there is a glaring issue with this strategy: There are two locked doors, but only one key. There must be some route that avoids one of the key doors, or maybe there's a way to get another key hidden somewhere. This single-use key can only be used once.

...Except that's actually a lie. It turns out there's a "bug" in the code responsible for disposing of the key. You might have noticed it already, if you tried the patched version of Nil Quest in this repository. In most places where an item is lost, the code calls the subroutine at 1653, which simply sets addresses 5912 and 5913 to zero. These addresses store the currently held item and a flag indicating whether the inventory is full, respectively. However, both key doors instead manually set 5913 to zero. This marks your inventory as empty, allowing you to pick up another item, but doesn't actually remove the key. Since the key doors only check 5912, the key can be used again, as long as you don't pick up another item before then. By getting rid of the goat in advance, it is possible to use the key on both the House and the Gate, allowing us to wield both the Sword and Shield in our fight against the dragon.

There's one last problem: This route takes 33 moves.  The remaining 9 moves aren't even enough to enter the secret room. We'll have to make some optimizations.

### Need for speed

It's clear that what we need now are some `extreme speedrunning strats`. There are two particular obstacles on our route that kill us, not directly, but by slowing us down.

The first is the bend in the Mountains. Traversing the mountain normally adds a single, deadly, move, but there's a way around this. If the world is rotated so that pressing the right arrow key corresponds to moving north, the extra move will be skipped. The world can be rotated in this manner by entering the tent and leaving with the up arrow key. You'll have to keep in mind that everything is now rotated a quarter turn clockwise. (It's also possible to skip the extra move using the rope from the Village, but you have to grab the rope, which kind of defeats the purpose.)

The second is the goat. Shooing the goat requires grabbing the axe and taking a second trip for the key, taking up many moves. There's a way around him, too. The code checks if `STAT(95)` is equal to zero, and if it is, the goat will kick you forward instead of back. `STAT(95)` in PICO-8 refers to the second hand on the system clock. This means we can sneak by the goat without shooing it if we wait until the first second of a minute. Ironically, waiting in the real world allows us to save moves in the game.

### Final route

At long last, we have everything needed to defeat the dragon. All that's left is to put it together.

One last thing to note is that the attack potion must be collected before the sword. Otherwise, the adventurer will get hungry, and, um, *eat* the sword sandwich. With this restriction, there is only one possible way to defeat the dragon in time. 

First, collect the key from the tent. Leave the tent going up. Use the key on the door in the village. Sneak by the goat when the system clock shows 0 seconds. Use the phantom key on the gate. Take the stone from the cliff. Take the elevator down. Move against the south wall of the Forest 12 times to enter the secret room. Drink the attack potion. Use the stone on the sword to get the sandwich. Take the shield from your house. Take the elevator back up. Finally, enter the dragon's lair and slay it with the sword.

If you do all this right, the quest will be complete without a move to spare. For some reason, the desktop version of PICO-8 refuses to load the victory screen. If you follow the same sequence of moves on the BBS version of the game, now completely blind, the victory screen will load, and you will be crowned the Reverse Hero!

## Appendix

A huge thank-you to [@thisismypassword](https://www.lexaloffle.com/bbs/?uid=29645) for making this fun puzzle. I also have to thank [@dddaaannn](https://www.lexaloffle.com/bbs/?uid=9599) for inadvertently helping me get started by posting their initial attempts in the comments, and while we're thanking people, I might as well thank Lexaloffle for making PICO-8 in the first place. 

This document is intended as a walkthrough for Nil Quest. It mostly follows the route I took, but I moved some parts around to make it flow better.

Figuring out the message command was actually one of the last things I did. I found the secret room by complete accident, while trying to experiment with the 43 move death condition by moving against random walls. I also created a map of the area, using lots of trial and error, plus careful reading of the assembly code. With only the message numbers to go by, I imagined that the tent was a strange forest of mirrors, and the goat was a mystical time-lock. Leaving the entire game to my imagination was kind of fun, although it made things very difficult. Finally seeing the real messages was like seeing an adaptation of a book that didn't quite match your imagination, but later realizing you liked the adaptation better.

Nil Quest uses a variety of interesting tricks to keep its solution hidden, including polymorphic code, so that the disassembly alone isn't enough to find the solution. As I was writing the disassembler, I realized one potential flaw: Since the instructions had variable lengths, it might be possible to thwart a primitive disassembler by padding the code with extra bytes. Thankfully, Nil Quest doesn't use this particular trick, and blindly going by instructions yields the full program. There are some data bytes mixed into the program at addresses 1582-1584, but since there are exactly three, the instructions stay aligned. I'm not entirely sure how I would change the disassembler to handle this situation. I would probably have to pay attention to what addresses are jump targets, to track what is real code and what is just misaligned garbage, or else produce simultaneous outputs corresponding to every possible reading of the machine code. Mixing this with polymorphism would create quite a tricky situation indeed.

I recently learned about an activity called "Security CTF", a form of capture-the-flag where the flags are pieces of data hidden in a computer. Some versions of the game involve hacking the opponent's system, but others instead have participants solve shared challenges involving things like cryptography or reverse engineering. These could be anything from cracking a cipher to tracing a redstone circuit in Minecraft. I haven't actually tried one of these events, but if you found Nil Quest interesting, you might like it.

No more text for you!