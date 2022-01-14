# A JavaScript port of the m8trix demo for MS-DOS

This is a JavaScript port of the [m8trix 8b demo](http://www.pouet.net/prod.php?which=63126) for MS-DOS,
originally written in x86 assembly by HellMood.

What makes this demo so fascinating is the fact
that the entire x86 machine code is just *7 bytes*.
That's right, the entire demo fits into a single register on a modern 64-bit CPU!
Even just the word "function" needs more space.

It used to take up 8 bytes originally, hence the name "m8trix".
The author later figured out a way to squeeze it down to 7 bytes
(read more about that at the link above).

## Notes on the JS version

Whereas the original demo directly writes into the video card's VGA text mode buffer
to make characters appear on the screen,
we use an HTML paragraph tag and continuously replace the content using a timer.
This means that our version is just an approximation in terms of speed and timing,
but the visual result is still fairly close.
Accurately recreating the exact timing of a DOS machine would add a lot of complexity without giving any further insights into the demo's algorithm,
so I decided to keep it simple and focus primarily on the character sequence generation.

The algorithm is exactly the same as the assembly version,
and is written in a way that closely replicates how the x86 instructions operate
rather than trying to reformulate the algorithm in idiomatic JavaScript.
Concretely, variable names match the names of the registers used,
and we use bit manipulation (bit shifts, AND and OR masking) to stuff two values
into a single variable like it is done with the registers in the Assembly version.

As a result,
the code may be somewhat obscure to a reader who's used to high-level programming.
I made sure to extensively comment the code to make it a bit easier to get into.
Ultimately, I think writing the code this way makes it easier to correlate it to the Assembly version,
and thus more valuable for understanding how the original code works.

If you find something confusing while reading the code,
feel free to open an issue asking for clarification,
or open a PR with suggested edits that might help make it easier to follow.

Also worth noting is that the code depends on some ECMAScript 6 features and thus might not work in old browsers.

## Original assembly code

For reference, the assembly code of the DOS version:

```asm
S:
  LES BX, [SI]
  LAHF
  STOSW
  XCHG CX, AX
  JMP S+1
```

One of the clever tricks used to keep the size down is the `JMP` at the end,
which has an offset of one byte from the start of the code (notice that it says `S+1`),
and thus actually jumps into *the middle of the first instruction* (`LES` has a two-byte encoding).
As it turns out, the resulting byte sequence (combination of the 2nd byte of `LES` and the one byte representing `LAHF`) is also a valid instruction,
namely `SBB`.
If we rewrite the code to make this "hidden" instruction visible, it becomes:

```asm
  LES BX, [SI]
  LAHF
S:
  STOSW
  XCHG CX, AX
  SBB AL, 9Fh
  JMP S
```

All the remaining tricks rely on the environment,
i.e. the CPU state that's set up by DOS when executing a `.COM` file.
In particular, the state of the registers `AX`, `CX`, `SI`, `DI`, and the CPU flags.

The key is that `SI` is set to `0x100`,
which happens to also be the memory address at which the program code itself is loaded.
This makes it so that the first instruction (`LES BX, [SI]`) reads the first 4 bytes of the demo's own code
and uses the 3rd and 4th byte to load the `ES` register.
So the 7 bytes don't just hold the code,
they also serve as data!
(The `LES` instruction also reads the 1st and 2nd byte into the `BX` register,
but it's not used afterwards and is thus irrelevant.)

The binary representation of the machine code is:

```
C4 1C 9F AB 91 EB F6
```

The bytes pointed to by `SI` are read as little-endian values,
so the sequence `9F AB` therefore results in setting `ES` to `0xAB9F`.
This conveniently causes the `STOSW` instruction's memory writes
to cover a range of addresses overlapping with the VGA text mode buffer address range.

Text mode video ram is mapped to the address range `B8000` to `B8FA0` on a DOS PC.
Because of segmented addressing,
the `STOSW` instruction combines the value of the `ES` register with the value of `DI` to determine the physical address for writing,
by essentially doing `ES << 8 + DI`.
The instruction also increments the `DI` register by two each time (since we are writing 2 bytes).
`DI` is a 16-bit register and wraps back around to `0` when incremented past `0xFFFF`,
but the `ES` register is unaffected by the wraparound.
This means that the demo's loop keeps writing into the memory region from `AB9F0` to `BB9EE`,
which includes the text mode screen buffer address range.

As for the CPU flags, these are (partially) loaded into `AH` via the `LAHF` instruction.
On DOS version 6.22, this causes `AH` to be set to `2`,
which conveniently happens to be the VGA text mode attribute flags byte needed for
dark green text on a black background.

Each `STOSW` instruction writes the two bytes stored in `AX`.
Due to little-endian encoding, it writes the low byte first (`AL`),
then the high byte (`AH`).
`AL` is initially 0 when the program starts.

The first loop iteration therefore writes a character value of `0` followed by an attribute value of `2`.
`AX` and `CX` are then swapped, so `AL` becomes 0xFF and `AH` becomes `0`.
The next `STOSW` thus writes an invisible character,
since an attribute value of `0` indicates black text with a black background.
On the next iteration, `AH` will be `2` again, etc.

If we further deobfuscate the code by making everything explicit, we end up with:

```asm
  MOV AX, 0200h
  MOV CX, 00FFh
  MOV DI, FFFEh
  MOV DX, AB9Fh
  MOV ES, DX
S:
  STOSW
  XCHG CX, AX
  SBB AL, 9Fh
  JMP S
```

Which is a bit more straightforward and also independent of the precise execution environment,
but also much less interesting and quite a bit bigger at 20 bytes.
