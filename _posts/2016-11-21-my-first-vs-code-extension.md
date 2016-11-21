---
title: My first Visual Studio Code extension for Kick Assembler C64
layout: post
---

I recently write my first extension for [Visual Studio Code](https://code.visualstudio.com/): *[Kick Assembler C64](https://marketplace.visualstudio.com/items?itemName=tomconte.kickassembler)*. If you are a little bit into retro computing/programming, you might know [Kick Assembler](http://www.theweb.dk/KickAssembler/) as one of the popular modern programs you can use to develop in 6502 assembly language for the Commodore 64. It is written in Java, which means it works on Mac, Linux and Windows.

The extension is really inspired by the [Kick Assembler package for Sublime Text](https://goatpower.org/projects-releases/sublime-package-kick-assembler-c64/) and is still nowhere near it in terms of functionality. It is mostly using the Kick Assembler syntax using its language file, which itself originates from [cbmeek's TextMate bundle](https://github.com/cbmeeks/6502KickAss-asm).

I have added a simple command, `Kick Assembler: Build` so that you can launch Kick Assembler directly from VS Code. Hopefully I will add more features as time permits :)
