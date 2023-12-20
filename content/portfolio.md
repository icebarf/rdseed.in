+++
title = "Projects"
description = "The projects I have worked on."
date = "2023-03-04"
aliases = ["projects", "works"]
author = "Icebarf"
+++

## [c8fontgen](https://github.com/icebarf/c8fontgen)
c8fontgen is a tiny pixel editor written in pure C using raylib, and raygui.
It is very minimal in functionality and was mostly created to test out the
capabilities of the previously stated libraries. A self contained executable
for linux based system exists for it in the releases section for you to try out.

## [dy-chip8-reborn](https://github.com/icebarf/dy-chip8-reborn)
It is an interpreter/emulator for the Chip8 virtual machine written in the C
programming language. It works with the development library [SDL2](https://www.libsdl.org/)
to provide graphics and audio support. This project is now archived.

## [Forth86](https://github.com/icebarf/FORTH86)
Forth86 is an interpreter for the 
[FORTH Programming Language](https://en.wikipedia.org/wiki/Forth_(programming_language))
written in x86_64 assembly language. It depends on my [libx86](#libx86) project, which provides a base for it.

## [libx86](https://github.com/icebarf/libx86)
libx86 is a small library in x86_64 assembly, written during my read of
Low-Level Programming by Igor Zhirkov. It is a simple include file that
provides a minimal yet, useful functionality while working with assembly.

## [mister8](https://github.com/icebarf/mister8)
mister8 is yet another chip8 interpreter implementation that behaves correctly.
My previous chip8 project had both behavioral and code quality issues. This project
tries to alleviate both of those. It uses raylib for input, monochrome graphics, and
beep audio.

## [Perfmode](https://github.com/icebarf/perfmode)
Perfmode is a fan/thermal policy control and keyboard LED backlight management
utility. It is written in C and works by writing to files exposed by
the Linux kernel module `asus-nb-wmi`. Perfmode has been under maintainenance
since 2021 and will continue to do so.

## [Telegram Channel to Discord Bridge](https://github.com/icebarf/tgchannel-discord-bridge)
This is a bridge bot that links a public telegram channel to a discord channel.
The bot interoperates with discord using [discord.py](https://discordpy.readthedocs.io/en/stable/)
and telegram using [telethon](https://docs.telethon.dev/en/stable/).
It's a specialised bot where public use was a secondary choice.
Minimal care was taken to make this bot suitable for public use by hosting your own
instance. This bridge is a result of a recreational activity turned into something useful for a discord server. It is currently being used in a server with ~4000 members whereby posting updates from 4 telegram channels to 3 discord channels.
