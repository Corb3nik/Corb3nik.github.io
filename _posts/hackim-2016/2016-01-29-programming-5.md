---
layout: post
title: Programming 5
category: HackIM-2016
---

## Description

Dont blink your Eyes, you might miss it. But the fatigue and exhaustion rules out any logic, any will to stay awake. What you need now is a slumber. Cat nap will not do. 1 is LIFE and 0 is DEAD. in this GAME OF LIFE sleep is as   important food. So... catch some sleep. But Remember...In the world of 10x10 matirx, the Life exists. If you SLOTH, sleep for 7 Ticks, or 7 Generation, In the game of Life can you tell what will be the state of the world?

  The world- 10x10

   0000000000,0000000000,0001111100,0000000100,0000001000,
   0000010000,0000100000,0001000000,0000000000,000000000

---

## The Challenge

  This was a fun challenge actually.

  Starting from the description, there are a couple of keywords that are useful :

  - 1 is LIFE
  - 0 is DEATH
  - GAME OF LIFE
  - The world- 10x10

  With this, we understand that the string of binary that is given to us is a 10x10 matrix.

    0000000000,
    0000000000,
    0001111100,
    0000000100,
    0000001000,
    0000010000,
    0000100000,
    0001000000,
    0000000000,
    000000000

  This reveals the number 7, which is also refered to by `7 ticks` and `7 generation` in the description.

  <br>
  As for the game of life part, it refers to [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway's_Game_of_Life), a computer game.

  At this point, all we have to do is find an implementation of `Conway's Game of Life`, input the matrix that was given to us, and step 7 times.

  Here is a javascript implementation we can use : [http://pmav.eu/stuff/javascript-game-of-life-v3.1.1/](http://pmav.eu/stuff/javascript-game-of-life-v3.1.1/)

  <br>
  Initial setup :

  ![start](/assets/img/hackim-2016/start.png "start")

  <br>
  7 steps (generations) later :

  ![end](/assets/img/hackim-2016/end.png "end")


  <br>
  Convert the blue squares into a 10x10 matrix :

    0000000000,
    0001100000,
    0001111010,
    0000001001,
    0000001010,
    0000000000,
    0000000000,
    0000000000,
    0000000000,
    0000000000

  **Flag** : 0000000000,0001100000,0001111010,0000001001,0000001010,
  0000000000,0000000000,0000000000,0000000000,0000000000
