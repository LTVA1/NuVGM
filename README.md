# NuVGM
A humble attempt to fix the issues of VGM file format...

**IMPORTANT!!!** I have no time nor programming skill and experience to implement the format in steady and nice fashion in some player app reference implementation (don't forget that a ripper is also needed). If smb could help with app and a playback library for others to use, it would be great.

If the format gets traction, though, I promise to implement Furnace tracker export routine with all the optimizations.

# Original ideas by tildearrow
my proposal for a successor to VGM (i'd call it NuVGM (thanks Iyatemu for the name idea), but feel free to use any name):
- a "list of chips". every chip has a few fields: type, clock rate, volume, panning, parameter length and parameters (parameters which are not recognized by a player which only supports an old version of NuVGM are ignored)
  - the type is 16-bit to ensure we can support up to 65536 chip types (should be more than enough)
  - the volume is 32-bit floating point
- a "logging rate" field
- an UTF-8-like chip selection scheme, when there are more than 128 chips:
  - e.g. if the "write to 8/8 register" command is 10, then we would do: `10 xx... yy zz` (where xx is the chip index, yy is the address and zz is the data)
  - if we want to access chip 0: `10 00 yy zz`
  - if we want to access chip 128: `10 C2 80 yy zz`
  - if we want to access chip 66376: `10 F0 90 8D 88 yy zz`
  - this way the format can store songs which use up to ~~2147483647~~ 1114111 (`0x10FFFF`) chips
- in order to remain compact, a few dedicated commands to write to the first four or so chips will be added
- different register write commands for different register widths
  - in the format addr/data: 8/8, 16/8, 16/16 (e.g. QSound), 32/8, 32/16, 32/32
- data blocks are chip-independent
- an "assign ROM data" command to bind a ROM to a chip, like so: `50 xx yyyy zz` (where xx is the chip index, yyyy is the data block and zz is the ROM type (for chips which can be assigned more than one, e.g. YM2610))
  - this also allows for some degree of bank switching (to be discussed further)
- PCM streams can use any data block, write to any address and allow for loop points
- an extensible, backward and forward compatible tag format like ID3v2 (should we call it GD3v2?)
  - tag data is UTF-8 (enough of Windows-specific UTF-16)

# More thoughts by me

The UTF-8 opcodes idea is great (however, I replaced it with just VLQ encoding), however, I have discussed the idea of block writes:
````
block header with chip id
address data
address data
address data
address data
block header with chip id
address data
address data
...
````
This helps with eliminating the opcodes, however, this can add more to the file size if we log interleaved writes to different chips (e.g. X68000 logs where YM2151 and OKI6258 data is heavily interleaved; thanks @ValleyBell!). So I stick to the tildearrow's idea.

````
a "logging rate" field
````
Well, more like a granularity option, and we assume that register writes are immediate (so any number of those takes exactly 0 time). Special commands may be used if we really want to rely on the delays.

PCM streams/chip RAM updates/special registers updates (like wavetables on Gameboy/N163) are to be isolated in special blocks and given a unique 16 or 32-bit identifier, and an opcode can reference those blocks by that index. This can be done in order for optimization program to be able to handle lots of different writes, storing each write only once (e.g. wavetable update on ES5503 is at least 256 bytes long, and if smb decides to use 4 sets of 256 packs of wavetables to simulate complex sound like C64 filtered waves or PWM VGM file will anyway be huge, but in the NuVGM we can store each wave only once, even if there are 1024 or more of them).

A special block where chip outputs can be assigned to output channels (kinda like Furnace's Patchbay). This is for chips that have more than two output channels (ES5503/5505/5506, OPL3, etc.). The player itself supports up to 256 audio channels, `0xff` being the NULL channel.

## The format MUST support all the chips that are even vaguely sound chips
This includes MOS Technology SID (both models, with customizable filter curve!), NES, etc. The format should not care if there is any other widespread existing format for the chip. If it chip-tunes, it gets added.

# Preliminary spec

All multibyte variables are stored little-endian, so `0x12345678` is written in file as `0x78 0x56 0x34 0x12`.

The file may have optional zlib compression.

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| `nVGM` format magic | ASCII string | 4 |  |
| EOF pointer | `uint32_t` | 4 | `(file_size - 8)` |
| Version | `uint32_t` | 4 | version |
| GD3 absolute offset  | `uint32_t` | 4 | things like music engine Hz rate, song/album cover, author, jump table for playlist if all the OST is stored inside one file etc. can be stored there idk |
| № of samples | `uint32_t` | 4 | total number of samples (sum of all wait commands) |
| File playback tick rate | `uint32_t` | 4 | For wait commands etc. |
| NuVGM data offset | `uint32_t` | 4 | absolute offset to the start of actual data |

Blocks of data follow.

Block structure:
| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Block name | ASCII string  | 4 |  |
| Block version | `uint32_t` | 4 |  |
| Block size | `uint32_t` | 4 | size of the block (excluding this field, block name and block version, so `(block_size - 12)`) |
| Block data | binary data | Block size | whatever inside the block |

Player must skip unknown blocks.

## Defined blocks

Here the block data of main blocks is described.

### Header

Block name is `FHDR`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Chip count | `uint32_t`  | 4 |  |

Then the array of chip settings follows, one per chip:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Total size of this chip's settings | `uint32_t` | 4 |  |
| Chip ID | `uint16_t` | 2 | Each chip type has a unique 16-bit signature. There can be unlimited amount of chips of the same type, but settings are individual per chip. |
| Volume | `float` | 4 |  |
| Panning | `float` | 4 | `-1.0` = left, `0` = center, `1.0` = right |
| Clock | `uint32_t` | 4 |  |
| Size of additional parameters section | `uint32_t` | 4 |  |
| Additional parameters section | binary data | Size of additional parameters section |  |

The section can contain whatever chip needs, like output routing, filter curve and chip sub-model (revision) for MOS Technology SID etc. You can also store extended panning data here.
Section may use the same block structure inside of itself.

Then the chip settings continue:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of RAM/ROM "units" | `uint8_t` | 1 |  |

Then the array of these "units" settings follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| RAM/ROM "unit" size | `uint32_t` | 4 |  |
| Additional data length | `uint32_t` | 4 |  |
| Data | binary data | Additional data length | Whatever info is needed (e.g. initial byte value to fill the memory with) |

The index of the entry in these arrays is used in data stream to reference it.

RAM/ROM units would have fixed indices for each chip type. For example, "YM2610 ADPCM-A" = unit value 0 and "YM2610 ADPCM-B" = unit value 1, "OKI6295 ROM" = unit value 0 (thanks @ValleyBell!).

### RAM/ROM data

Block name is `ROAM`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of RAM/ROM write sections | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Size of the section | `uint32_t` | 4 |  |
| Data type | `uint16_t` | 2 |  |
| Compression type | `uint16_t` | 2 | compression is applied to the raw data which originally was in format described by the field above |
| Data | binary data | Size of the section |  |

Special values of data/compression type may be reserved to signal that the range of RAM/ROM must be filled with 0's or a particular byte/repeating byte sequence. In this case the actual data may be omitted.

### PCM stream

Block name is `NPCM`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of PCM streams | `uint16_t` | 2 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| PCM stream total size | `uint32_t` | 4 |  |
| Flags | `uint8_t` | 1 | See below: |

````
(bit 0 is LSB, bit 7 is MSB)
bit 0 - if we ignore loop points (loop just from beginning to end)
bit 1 - enable/disable loop
bit 2 - ping-pong loop
bit 3 - play in reverse (loop points not valid here)
````

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Data type | `uint16_t` | 2 |  |
| Compression type | `uint16_t` | 2 | compression is applied to the raw data which originally was in format described by the field above |
| Data | binary data | Size of the section |  |

A special opcode is used for manipulating PCM streams (see below).

### Main logged data block

Block name is `NLGD`

The block contains data in form of opcodes and their operands stream, much like the VGM.

## Opcodes

In the main logged data block there is a stream of commands. Each command has an opcode as its first byte, then one or more bytes of data follow. 

Opcode operands are given in "nibble-notation". For example, if opcode operands are `aa pp dddd xy zzzzzzzz`, this means that `aa` is 8-bit value, as well as `pp`, `dddd` is 16-bit value, `xy` byte holds two parameters, `x` and `y`, which occupy their respective nibbles, and `zzzzzzzz` is 32-bit value.

A special case is `aa...` operand. If written like this, it means that this field (`aa`) is VLQ standard code value (see [spec](https://en.wikipedia.org/wiki/Variable-length_quantity)).

There are some shortened opcodes for chips with indices 0 to 7. These are introduced because they are expected to be the most common ones, and shaving one byte off them can noticeably reduce file size.

Player must halt playback if it encounters an unknown opcode. Player may throw a warning/error message upon encountering it.

Opcodes are described there:

| Opcode | Operands | Description |
| ------------- | ------------- | ------------- |
| `0x00` |  | NOP. Do nothing and immeditely go to next opcode |
| `0x01` | `dddddddd bb cc.. oooooooo` | Write data block `dddddddd` to ROM/RAM "unit" `bb` of chip `cc..` with starting offset `oooooooo` (for ROM it is used on the init phase so we can fill ROM with only the required data instead of storing the whole ROM image) |
| `0x02` | `bb cc..` | (only for banking) enable ROM/RAM "unit" `bb` for chip `cc...` (make chip use the bank `bb`, if you will) |
| `0x03` | `tt...` | Wait `tt...` samples |
| `0x10` | `aa dd` | Write data `dd` to register with address `aa` of chip `0` |
| `0x11` | `aa dd` | Write data `dd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x18` | `cc... aa dd` | Write data `dd` to register with address `aa` of chip `cc..` |
| `0x20` | `aa dddd` | Write data `dddd` to register with address `aa` of chip `0` |
| `0x21` | `aa dddd` | Write data `dddd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x28` | `cc... aa dddd` | Write data `dddd` to register with address `aa` of chip `cc...` |
| `0x30` | `aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `0` |
| `0x31` | `aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x38` | `cc... aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `cc...` |
| `0x40` | `aaaa dd` | Write data `dd` to register with address `aaaa` of chip `0` |
| `0x41` | `aaaa dd` | Write data `dd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x48` | `cc... aaaa dd` | Write data `dd` to register with address `aaaa` of chip `cc...` |
| `0x50` | `aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `0` |
| `0x51` | `aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x58` | `cc... aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `cc...` |
| `0x60` | `aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `0` |
| `0x61` | `aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x68` | `cc... aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `cc...` |
| `0x70` | `aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `0` |
| `0x71` | `aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x78` | `cc... aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `cc...` |
| `0x80` | `aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `0` |
| `0x81` | `aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x88` | `cc... aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `cc...` |
| `0x90` | `aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `0` |
| `0x91` | `aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x98` | `cc... aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `cc...` |
| `0xan` |  | wait `n+1` ticks, `n` can range from 0 to 15 |
| `0xb0` |  | PCM stream manipulation (see below) |

### PCM stream control

This opcode defines a set of sub-commands for controlling the PCM stream.

Pattern: `B0 [sub-command] [sub-command params]`.

Setup stream:
````
B0 01 ss ss cc... aa aa aa aa
````
`ss ss` is stream number, `cc...` is chip number and `aa aa aa aa` is register address into which samples are streamed.

Set stream data entry:
````
B0 02 ss ss ii ii
````
`ss ss` is stream number, `ii ii` is PCM writes block entry index (which actually may contain multiple samples).

Set stream frequency:
````
B0 03 ss ss ff ff ff ff
````
`ss ss` is stream number, `ff ff ff ff` is frequency in Hz with which writes are happening.

Set stream start offset:
````
B0 04 ss ss oo oo oo oo
````
`ss ss` is stream number, `oo oo oo oo` is start offset in PCM writes block entry.

Set stream data length:
````
B0 05 ss ss ll ll ll ll
````
`ss ss` is stream number, `ll ll ll ll` is data length in PCM writes block entry (the start of data is start offset specified in the previous command). If length is `0` we play the whole same, from start offset to the end of the PCM writes block entry.

Set stream loop/playback mode:
````
B0 06 ss ss ll
````
`ss ss` is stream number, `ll` is loop mode:
````
(bit 0 is LSB, bit 7 is MSB)
bit 0 - no loop
bit 1 - normal loop
bit 2 - ping-pong loop
bit 3 - play in reverse
````

Set stream loop point:
````
B0 07 ss ss ll ll ll ll
````
`ss ss` is stream number, `ll ll ll ll` is loop point relative to PCM writes block entry data start.

Start stream:
````
B0 08 ss ss
````
`ss ss` is stream number.

Stop stream:
````
B0 09 ss ss
````
`ss ss` is stream number.

Other opcodes:

| Opcode | Operands | Description |
| ------------- | ------------- | ------------- |
| `0xf0` | `ll` | Marks begin or end of the loop. If `ll` is 0, it's loop begin, otherwise it means "loop to loop begin point `ll` times". Nested loops are possible. |
| `0xf1` | `aa aa aa aa` | jump to absolute offset `aa aa aa aa` |
| `0xff` |  | Halt (stop) playback |

# Future plans

Afaik many songs are structured in a way that uses instruments, them being a repeating initialization of registers. This is particularly obvious with FM chips like OPNx/OPM, there even exist some programs to rip the patches from VGM files afaik.

Thus we have a repeating initialization of the same registers (or registers on different channel with the same instrument... Some chips even have the register set for only one channel and switch them via special register... doesn't differ a lot actually) over and over. This leads to the idea of two new opcodes.

`jsr` opcode would take an absolute address of the start of the subroutine and jump there. The execution would carry on until `rts` opcode is met. At the `rts` opcode the player jumps to the next opcode after the `jsr` it used to jump to subroutine. Nested subroutines would be possible.

*jsr - **j**ump **s**ub**r**outine

*rts - **r**e**t**urn from **s**ubroutine

When exporting from the tracker, it (and loop opcode described earlier) would allow to optimize the file even more by just making an entire patterns subroutines. With trackers like Furnace tracker where order is a matrix it's not so easy, but repeating matrix rows could be encoded like this too.
