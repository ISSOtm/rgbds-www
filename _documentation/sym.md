---
layout: default
permalink: /docs/sym
title: Sym file reference
description: GB .sym file reference
---

# Game Boy symbol (`.sym`) file reference

## Purpose & Context

Game Boy ROM files contain pure binary data, but no metadata.
In particular, symbol information is lacking when debugging or reverse-engineering such ROMs.
Sym files aim to be companions to Game Boy ROM files that provide exactly that: symbol information.

Sym files have been around for a long time (the [no$gmb](http://problemkaputt.de/gmb.htm) emulator supported them, so it is safe to assume since at least the early 2000's), but the format was always loosely defined.
This living document aims to standardize the format and usage of sym files, so that consumers know what to expect.
Modifications, etc. should be suggested by [opening an issue](https://github.com/gbdev/rgbds-www/issues) or [starting a discussion](https://github.com/gbdev/rgbds-www/discussions) on this website's source repository, whichever is more appropriate.

Extensions to this format are known to exist ([example](https://github.com/mattcurrie/mgbdis/#symbol-files)), but they tend to be specific to a certain use case, and are generally a superset of this.
Keep in mind that some sym files are written manually, such as when disassembling ROMs, so it is recommended for consumers to be lenient.

Note also that [wla-dx's sym files](https://wla-dx.readthedocs.io/en/latest/symbols.html) are much more complex (and complete) than this format.
RGBDS aims to keep sym files simple, and instead chooses to output other data in different files, such as [map files](map).

## File format specification

Line endings can be either LF-only or CRLF, though the former is preferred.
The character encoding must be UTF-8 (without BOM), however it is strongly encouraged to stick to ASCII characters for portability.
The NUL character (\\0, U+0000) shall not appear anywhere in the file.
An implementation encountering one must output a diagnostic if it can, and may then ignore it, the line, the whole file, or reject the file outright.

Especially since they are intended to be somewhat human-readable, sym files can contain comments.
Comments begin with a semicolon anywhere in a line, all the way to the end of that line.
Semicolons cannot be escaped, as they are not valid in any part of entries.

After removing the comment if any, the line shall be split into non-empty tokens separated by ASCII spaces (U+0020).
Implementations may also split on tabs (\\t, U+0009), but this is not portable to some older implementations.

A line without any tokens shall be silently ignored, to permit empty lines.

A line with only one token shall be ignored, being reserved for future extensions.
Encountering one may produce a warning.

Otherwise, the line encodes a single entry: the first token provides its location; the second token provides its symbol's name; and further tokens may provide extra metadata.
If any token is not understood by the implementation, it (and possibly the whole line) shall be ignored, possibly with a warning.

The location shall be of the form `<bank>:<address>`, both being case-insensitive hexadecimal numbers, possibly zero-padded, but without any prefix.
Some implementations allow the bank to be the case-insensitive keyword `BOOT` to specify symbols mapped to the boot ROM.
The bank may be omitted, but the results are implementation-defined.

The symbol name can contain any characters but spaces, tabs, ASCII control characters, NULs, and semicolons.
If a name contains exactly one period, and the portion of the name up to and not including the period matches a previous symbol and no other symbols without periods are defined between that previous symbol and the current one, then the current symbol can be considered a local symbol and be shortened in presentation to the local portion (i.e., the period and everything that follows).

## Precautions

Be advised that it may sometimes be impossible to detect which memory region a symbol belongs to: for example, `0:A000` could be at the end of VRAM, or at the beginning of SRAM.

Several symbols can share the same bank+address, such as in the above, but also originating from the same memory region.
Unions, in particular, facilitate this.

Entries may be unsorted (RGBLINK pre-0.4.0 output symbols as it found them); if they are, they may be by bank then address (typically from [`sort(1)`](https://man.openbsd.org/sort.1) output), by "bank+address", i.e. sorted by memory region, then bank, then address (many sorting scripts, and the behavior of RGBLINK [since v0.4.0](https://github.com/gbdev/rgbds/releases/tag/v0.4.0)).

Note that banking for addresses `$0000-7FFF` is defined by the ROM's mapper, so flexibility is again advised: the bank number for `$0000-3FFF` may not be 0 ([MBC1M, MMM01, ...](https://gbdev.io/pandocs/#multicart-mbcs)), the bank number for `$4000-7FFF` may be 0 ([mapper-less ROMs](https://gbdev.io/pandocs/#no-mbc)), etc.
Note also that while most ROMs never have more than 256 banks for anything, at least one commercial GBC game ([Densha De Go! 2](https://github.com/gbdev/awesome-gbdev/blob/b8ead97724757651e393be951bfac238ab1f4d64/CartridgeList.csv#L496)) has more than 256 ROM banks, and homebrew mappers (such as the [TPP1](https://github.com/TwitchPlaysPokemon/tpp1)) may contain much more.

Sym files generally do not document hardware registers (`$FF00-FF7F` and `$FFFF`), so implementations are encouraged to provide separate support for those.
[hardware.inc](https://github.com/gbdev/hardware.inc) is a good starting point.

## Usage

**This section merely describes typical behavior.**

Most implementations, mainly emulators, only accept a ROM file as their input, and do not provide a way to manually pick a sym file.
Typically, those implementations look for a sym file in the same directory and with the same filename as the supplied ROM, except the file extension (usually `.gb`, `.gbc` or sometimes `.sgb`; although `.dmg` and `.bin` have been observed in the wild) is replaced with `.sym`.
