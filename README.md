# NuVGM
A humble attempt to fix the issues of VGM file format...
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

The UTF-8 opcodes idea is great, however, I have discussed the idea of block writes:
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

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| `nVGM` format magic | ASCII string | 4 |  |
| EOF pointer | `uint32_t` | 4 | `(file_size - 8)` |
| Version | `uint32_t` | 4 | version |
| GD3 offset  | `uint32_t` | 4 | relative; things like music engine Hz rate, song/album cover, author, jump table for playlist if all the OST is stored inside one file etc. can be stored there idk |
| № of samples | `uint32_t` | 4 | total number of samples (sum of all wait commands) |
| File playback tick rate | `uint32_t` | 4 | For wait commands etc. |
| NuVGM data offset | `uint32_t` | 4 | absolute offset to the start of actual data (?? maybe unnecessary?) |

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
| Chip list size | `uint32_t`  | 4 |  |
| Chip list | `uint16_t` array  | 2 * Chip list size | Each chip type has 16-bit signature. There can be unlimited amount of chips of the same type, but settings are individual per chip. |

Then the array of chip settings follows, one per chip:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Volume | `float` | 4 |  |
| Panning | `float` | 4 | `-1.0` = left, `0` = center, `1.0` = right |
| Clock | `uint32_t` | 4 |  |
| Version of additional parameters section | `uint32_t` | 4 |  |
| Size of additional parameters section | `uint32_t` | 4 |  |
| Additional parameters section | binary data | Size of additional parameters section |  |

The section can contain whatever chip needs, like output routing, filter curve and chip sub-model (revision) for MOS Technology SID etc. You can also store extended panning data here.

### RAM data

Block name is `NRAM`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of RAM write sections | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| RAM offset (including bank info) | `uint32_t` | 4 |  |
| Size of the section | `uint32_t` | 4 |  |
| Data type | `uint16_t` | 2 |  |
| Compression type | `uint16_t` | 2 | compression is applied to the raw data which originally was in format described by the field above |
| Data | binary data | Size of the section |  |

Then the declaration of RAM "units" follows. These can be assigned to different chips if it is supported by emulators.

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of RAM "units" | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| RAM "unit" size | `uint32_t` | 4 |  |
| Additional data length | `uint32_t` | 4 |  |
| Data | binary data | Additional data length | Whatever info is needed (e.g. initial byte value to fill the memory with) |

The index of the entry in these arrays is used in data stream to reference it, so player must build up some kind of LUT when scanning the file to make it possible.
Special values of data/compression type may be reserved to signal that the range of RAM must be filled with 0's or a particular byte/repeating byte sequence. In this case the actual data may be omitted.

### ROM images (or parts of them; includes banks and whatever)

Block name is `NROM`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of ROM sections/ROMs | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| ROM offset (including bank info) | `uint32_t` | 4 |  |
| Size of the section | `uint32_t` | 4 |  |
| Data type | `uint16_t` | 2 |  |
| Compression type | `uint16_t` | 2 | compression is applied to the raw data which originally was in format described by the field above |
| Data | binary data | Size of the section |  |

Then the declaration of ROM "units" follows. These can be assigned to different chips if it is supported by emulators.

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of ROM "units" | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| ROM "unit" size | `uint32_t` | 4 |  |
| Additional data length | `uint32_t` | 4 |  |
| Data | binary data | Additional data length | Whatever info is needed (e.g. initial byte value to fill the memory with) |

The index of the entry in these arrays is used in data stream to reference it, so player must build up some kind of LUT when scanning the file to make it possible to assign different ROMs to different chips.
Special values of data/compression type may be reserved to signal that the range of ROM must be filled with 0's or a particular byte/repeating byte sequence. In this case the actual data may be omitted.

### Loop points

Block name is `NLPS`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of loop points | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Loop begin absolute offset | `uint32_t` | 4 |  |
| Loop end absolute offset | `uint32_t` | 4 |  |

A special opcode is used for halting the playback (see below).

### PCM stream

Block name is `NPCM`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of PCM streams | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| PCM stream total size | `uint32_t` | 4 |  |
| PCM stream loop sections | `uint32_t` | 4 |  |

The the array of specifying loop sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Loop begin absolute offset | `uint32_t` | 4 |  |
| Loop end absolute offset | `uint32_t` | 4 |  |

A special opcode is used for manipulating PCM streams (see below).

### Main logged data block

Block name is `NLGD`

The block contains data in form of opcodes and their operands stream, much like the VGM.

## Opcodes

In the main logged data block there is a stream of commands. Each command has an opcode as its first byte, then one or more bytes of data follow. 

Opcode operands are given in "nibble-notation". For example, if opcode operands are `aa pp dddd xy zzzzzzzz`, this means that `aa` is 8-bit value, as well as `pp`, `dddd` is 16-bit value, `xy` byte holds two parameters, `x` and `y`, which occupy their respective nibbles, and `zzzzzzzz` is 32-bit value.

A special case is `aa...` operand. If written like this, it means that this field (`aa`) is UTF-8 standard "symbol" which Unicode code point is actually a value, so it has variable length (1-4 bytes, see UTF-8 standard).

There are some shortened opcodes for chips with indices 0 to 7. These are introduced because they are expected to be the most common ones, and shaving one byte off them can noticeably reduce file size.

Player must halt playback if it encounters an unknown opcode. Player may throw a warning/error message upon encountering it.

Opcodes are described there:

| Opcode | Operands | Description |
| ------------- | ------------- | ------------- |
| `0x01` | `cc... bbbbbbbb` | Assign ROM `bbbbbbbb` to chip `cc...` |
| `0x02` | `dddddddd bbbbbbbb` | Write ROM data block `dddddddd` to ROM `bbbbbbbb` (used so on the init phase we can fill ROM with only the required data instead of storing the whole ROM image) |
| `0x03` | `cc... bbbbbbbb` | Assign RAM "unit" `bbbbbbbb` to chip `cc...` |
| `0x04` | `dddddddd bbbbbbbb` | Write RAM data block `dddddddd` to RAM "unit" `bbbbbbbb` |
| `0x05` | `tt...` | Wait `tt...` samples |
| `0x10` | `cc... aa dd` | Write data `dd` to register with address `aa` of chip `cc...` |
| `0x11` | `aa dd` | Write data `dd` to register with address `aa` of chip `0` |
| `0x12` | `aa dd` | Write data `dd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x18` | `aa dd` | Write data `dd` to register with address `aa` of chip `7` |
| `0x20` | `cc... aa dddd` | Write data `dddd` to register with address `aa` of chip `cc...` |
| `0x21` | `aa dddd` | Write data `dddd` to register with address `aa` of chip `0` |
| `0x22` | `aa dddd` | Write data `dddd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x28` | `aa dddd` | Write data `dddd` to register with address `aa` of chip `7` |
| `0x30` | `cc... aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `cc...` |
| `0x31` | `aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `0` |
| `0x32` | `aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `1` |
| `...` | `...` | `...` |
| `0x38` | `aa dddddddd` | Write data `dddddddd` to register with address `aa` of chip `7` |
| `0x40` | `cc... aaaa dd` | Write data `dd` to register with address `aaaa` of chip `cc...` |
| `0x41` | `aaaa dd` | Write data `dd` to register with address `aaaa` of chip `0` |
| `0x42` | `aaaa dd` | Write data `dd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x48` | `aaaa dd` | Write data `dd` to register with address `aaaa` of chip `7` |
| `0x50` | `cc... aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `cc...` |
| `0x51` | `aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `0` |
| `0x52` | `aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x58` | `aaaa dddd` | Write data `dddd` to register with address `aaaa` of chip `7` |
| `0x60` | `cc... aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `cc...` |
| `0x61` | `aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `0` |
| `0x62` | `aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x68` | `aaaa dddddddd` | Write data `dddddddd` to register with address `aaaa` of chip `7` |
| `0x70` | `cc... aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `cc...` |
| `0x71` | `aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `0` |
| `0x72` | `aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x78` | `aaaaaaaa dd` | Write data `dd` to register with address `aaaaaaaa` of chip `7` |
| `0x80` | `cc... aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `cc...` |
| `0x81` | `aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `0` |
| `0x82` | `aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x88` | `aaaaaaaa dddd` | Write data `dddd` to register with address `aaaaaaaa` of chip `7` |
| `0x90` | `cc... aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `cc...` |
| `0x91` | `aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `0` |
| `0x92` | `aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `1` |
| `...` | `...` | `...` |
| `0x98` | `aaaaaaaa dddddddd` | Write data `dddddddd` to register with address `aaaaaaaa` of chip `7` |
| `0xan` |  | wait `n+1` samples, `n` can range from 0 to 15 |
| `0xb0` |  | PCM stream manipulation (see below) |

### PCM stream control

This opcode defines a set of sub-commands for controlling the PCM stream.

Pattern: `B0 [sub-command] [sub-command params]`.

Setup stream control:
````
B0 01 ss ss cc... aa aa aa aa dd dd dd dd
````
`ss ss` is stream number, `cc...` is chip, `aa aa aa aa` is address and `dd dd dd dd` is the data you put there. This usually puts some chip channel into PCM mode or whatever. Special values may be used if extra action is needed to prepare the chip to PCM stream.

Set stream data:
````
B0 02 ss ss ii ii ii ii ll oo oo oo oo
````
`ss ss` is stream number (not the number which is index in PCM stream block! it is like this so you can do e.g. interleaved writes from the same data section), `ii ii ii ii` is PCM writes block index, `ll` is step (1 if you just go byte by byte, 2 if you skip every other byte etc.), `oo oo oo oo` is an offset in the PCM write array from which playback will start.

Set stream frequency:
````
B0 03 ss ss ff ff ff ff
````
`ss ss` is stream number, `ff ff ff ff` is frequency in Hz with which writes are happening.

Start stream:
````
B0 04 ss ss cc
````
`ss ss` is stream number, `cc` is control flags:
````
(bit 0 is LSB, bit 7 is MSB)
bit 0 - if we ignore loop points
bit 1 - loop (loop points not valid here)
bit 2 - ping-pong loop (loop points not valid here)
bit 3 - play in reverse (loop points not valid here)
````

Stop stream:
````
B4 05 ss ss
````
`ss ss` is stream number.

Set stream loop begin:
````
B4 06 ss ss ll ll ll ll
````
`ss ss` is stream number, `ll ll ll ll` is loop begin offset

Set stream loop end:
````
B4 07 ss ss ll ll ll ll
````
`ss ss` is stream number, `ll ll ll ll` is loop end offset

Set stream loop mode:
````
B4 08 ss ss mm
````
`ss ss` is stream number, `ll ll ll ll` is loop mode:
````
0 = no loop
1 = normal loop
2 = ping-pong loop
````

Other opcodes:

| Opcode | Operands | Description |
| ------------- | ------------- | ------------- |
| `0xff` |  | Halt (stop) playback |
