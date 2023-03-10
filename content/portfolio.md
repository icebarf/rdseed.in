+++
title = "Projects"
description = "The projects I have worked on."
date = "2023-03-04"
aliases = ["projects", "works"]
author = "Amritpal Singh"
+++

## [dy-chip8-reborn](https://github.com/icebarf/dy-chip8-reborn)
It is an interpreter/emulator for the Chip8 virtual machine written in the C
programming language. It works with the development library [SDL2](https://www.libsdl.org/)
to provide graphics and audio support. This project is now archived in favor
of my in-development project - [based-chip8-pp](https://github.com/icebarf/based-chip8-pp), using C++. 

## [Forth86](https://github.com/icebarf/FORTH86)
Forth86 is an interpreter for the 
[FORTH Programming Language](https://en.wikipedia.org/wiki/Forth_(programming_language))
written in x86_64 assembly language. It depends on my [libx86](#libx86) project, which provides a base for it.

## [libx86](https://github.com/icebarf/libx86)
libx86 is a small library in x86_64 assembly, written during my read of
Low-Level Programming by Igor Zhirkov. It is a simple include file that
provides a minimal yet, useful functionality while working with assembly.

## [Perfmode](https://github.com/icebarf/perfmode)
Perfmode is a fan/thermal policy control and keyboard LED backlight management
utility. It is written in C and works by writing to files exposed by
the Linux kernel module `asus-nb-wmi`. Perfmode has been under maintainenance
since 2021 and will continue to do so.
