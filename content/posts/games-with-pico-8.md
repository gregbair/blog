---
title: My experience with PICO-8
tags:
- pico-8
slug: games-with-pico-8
date: 2020-10-31
---

![Blackjack](/blackjack.p8.png)

This weekend, I decided to try something different - programming a game in some esoteric environment. Since I don't have a [Commodore 64](https://en.wikipedia.org/wiki/Commodore_64), I went with the [PICO-8](https://www.lexaloffle.com/pico-8.php), and man did I have fun.

It reminded me a lot of programming in BASIC back in the day, typing in games from magazines and books.

## Overview of features

* Built-in sprite and SFX editors
* Built-in editor for full retro experience
* Can edit files outside of the VM as well

## Pros

I really liked at first the idea of programming inside the machine itself, but that quickly got tedious for me. Fortunately, the .p8 file is just a text file. The code language is a subset of [Lua](https://www.lua.org/). I'd never coded in Lua before, but it's a fun little scripting language.

The built-in sprite editor is really nice. Has all the basic things you'd want in a sprite editor, the ability to change the size of the sprites, etc. You can even export the sprites to a PNG!

## Cons

I felt like learning the system with just the basic docs was quite difficult. I ended up getting a book, [_Game Development with PICO-8_](https://mboffin.itch.io/gamedev-with-PICO-8-issue1) that really helped. I also bookmarked the [system manual](https://www.lexaloffle.com/pico8_manual.txt) which has good API docs in it.

There's also no debugging method other than `print()`, so if you rely on debugging in an IDE or at the command line, tough. Most BASIC interpreters back in the day didn't either, so no big loss there.

## Ok, so where's the game?

I built a blackjack game! It's super simple, aces are always 1, there's no splitting, no insurance, nothing like that. It was really fun to build.

However, occasionally when starting/restarting the game, we get a nil reference to either one of the player's cards or one of the dealer's cards. Not sure what's going on there, but I suspect I'm not doing some initialization correctly.

* [P8 file](/blackjack.p8)
* [Play in browser](/blackjack.html)
* [PNG](/blackjack.p8.png) (Yes, you can export it as a PNG, and import that PNG into PICO-8!)
