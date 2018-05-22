# 2048
This is a clone of a repository of the same name by Gabriele Cirulli (see fork info above for link).

The game can be played [here](http://www.jakechitel.com/2048).

The primary purpose of this fork is to serve as a platform for automated gameplay. This mode will be toggleable.

## Strategy

The automation strategy follows a manual strategy that I call the "snake". It involves focusing the "head square" (the square with the largest number) at one corner (I prefer the bottom left) and tracing a descending, ideally uninterrupted, zig-zag from there until the lower numbers. Whenever there is a complete snake from the head to the tail, the snake will collapse into the head and the process will begin again.

This strategy can ideally follow until the theoretical limit of the game:

```
2/4    8     16    32
512    256   128   64
1024   2048  4096  8192
131072 65536 32768 16384
```

It is possible to reach 131072 because 4's can spawn, and if you had a full and perfect 65536 snake with 2 4's at the end, you can collapse that to 131072. But that would involve quite a bit of chance. The highest I've been able to go manually is (I believe) 8192 with a 4096 and 2048 to go with it.

### Phases

There are several phases involved with this strategy:

#### Fixing

The initial goal of the strategy is to begin building the bottom row. You generally want to focus a 2 or 4 on the bottom left, then start piling squares on the bottom row and collapsing leftward. The result:

```
x  x x x
x  x x x
x  x x x
16 8 4 2
```

Filling up the row opens you up to being able to move right as well as left. During this phase, you want to avoid moving right or up at all costs, but at times it is unavoidable, in which case the priority is to move right if possible, and up as a last resort.

NOTE: I still do not have a strategy for avoiding this necessity.

#### Building

The fixing phase creates the start of the snake and opens up the main mode of gameplay: building. This is a recursive phase because you are always building to duplicate the current end of the tail, then collapsing. Collapsing will work toward duplicating the next segment of the tail, and so on until there is an uninterrupted chain from the head to the end of the tail, then you do one more collapse to increment the head and start over.

The trick here is to prevent moving the current snake, because moving is likely to cause unintended interruptions that need to be handled. The ideal case is to be able to build only in the two directions that will not move the tail: down and either left or right depending where the tail is. Once the tail end has been duplicated, collapsing can begin. If an interruption is introduced, then recovery must begin.

#### Collapsing

Collapsing is the dream state. You wish you could always be collapsing, but that's just a small part of the game. Once you have a tail duplicate next to the end of the current tail, you can collapse the tail as far as the unbroken chain currently goes. Eventually there will be a collapsing phase that goes all the way to the head.

The difficulty with collapsing is to make sure you don't leave yourself in a bad state during the process. You need to be able to start building again right away, and there's not much more frustrating than doing a full tail collapse and then realizing you have to move right or up. This involves making sure that at least the bottom row is fixed during the whole collapse, but there are still some squares left above that can move right.

#### Recovery

This phase is (yet to be proven) unavoidable. Eventually you are going to find yourself in a situation where you have to move a large square to a place where you might wind up introducing an interruption. The goal here is to get yourself back to where you were, and there are two ways to do that: secondary snake and restoration.

##### Secondary Snake

The secondary snake strategy is good for recoveries in the early game, when interruptions aren't quite as devastating. Effectively you accept the interruption as a new head and start building on that, ignoring your previous snake temporarily. Eventually, the new head will be a duplicate of a part of the original snake, and you can collapse it into the original snake, then proceed as normal.

Taking this strategy is dangerous in the late game because the ignored squares are a waste of space, and you are likely to fill up the board if your goal number is large enough. The benefit of this strategy is that it is not very risky. It simply involves putting the original game on pause and starting a new one that will eventually merge back into the original.

##### Restoration

The restoration strategy is far riskier than the secondary snake, but it is more valuable in the late game because it can avoid filling up the board quite as easily. This involves trying to recreate the situation that caused the interruption in order to slide the misplaced head back into its original position. In order for this to be successful, you need to fix the head along an alternate axis and attempt to create a situation where you can force an opening. This involves filling up quite a bit of the board, but you usually end up doing it with much smaller numbers that are quickly collapsed once the situation is resolved.

This is dangerous because the spawn square is random, and creating the opening might just end up filling it up again. The good news is that you won't be in a worse situation than you started in, so you can usually just try again. The extra danger is that while this strategy can avoid filling the board in the long term, it can fill it up in the short term if you aren't careful. It is often prudent to keep an eye on what is filling up the board, and work on some collapsing if it is getting too full.

### Gameplay and Analysis

The gameplay will be the tricky part starting out. When playing manually, you have a combination of visualizing the board and thinking about what will happen if you move one way or another, as well as the current goal. The benefit of the automation is that it is not prone to stupid mistakes that happen when playing too quickly.

#### Analyzing Available Moves

A move is valid when any square has either an empty square or a duplicate of itself in the desired direction.

A move is prudent when it is both valid and moves you toward your current goal. The automation will have the ability to visualize the board after a given move, or even after a series of moves. Prediction is difficult though, because the automation can't fully predict where new squares will end up. Initially we will just have 1-move lookahead, but we may be able to implement multi-move lookahead with probability in time.

#### Process

There are four primary phases of the game, detailed above. At any given point, the automation will be in one of the phases. It will be useful to consider gameplay as a stack of goals. Initially you have one goal: to create a snake with the largest possible head, necessarily producing the largest possible number. The automation will have a standard procedure for meeting that goal:

1. Start the snake if it isn't already
2. Fix the bottom row
3. Build the snake to whatever the current end is
4. Collapse, keeping the bottom row fixed
5. Repeat from 3

So steps 1 and 2 will be what we call the "bootstrapping phase", where the goal is to get to the standard loop on the way to the main goal, which will never change. Once bootstrapping is done, we will start building a standard goal stack. Assuming that the current head is 16, the goal will be to make that a 32. To do that, we need to put a 16 next to it in the snake. If that number is currently an 8, then we need to put an 8 next to it. If that number is currently a 4, then we need to put a 4 next to it. If that number is currently a 2, then we need to put a 2 next to it. At that point, if the next square is empty, we need to get a 2 to that location. More than likely, the nearest 2 is 2 moves away (unless it's in line, in which case it's just one guaranteed move away).

We will need to analyze to determine what will happen when we move. The biggest benefit of automation is the ability to take things "slow". We can consider not just the square under focus, but all squares. We can know exactly where they will end up and plan ahead. If a move toward the current goal will potentially wind us up in a sticky situation, we can insert some safer moves in between.

If we ever reach a recovery mode point, that will effectivly involve peeling a bunch of goals off the stack and replacing them with a recovery goal. In the worst case, this will involve replacing every non-static goal. In the best case, we might just replace the current goal. Light recovery mode is likely to be very common because the building phase has side effects that almost always involve creating interruptions. I have yet to devise a way to handle these manually, but automation might have a better time.

TBA...