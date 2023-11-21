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
- DAC streams can use any data block, write to any address and allow for loop points
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

DAC streams/chip RAM updates/special registers updates (like wavetables on Gameboy/N163) are to be isolated in special blocks and given a unique 16 or 32-bit identifier, and an opcode can reference those blocks by that index. This can be done in order for optimization program to be able to handle lots of different writes, storing each write only once (e.g. wavetable update on ES5503 is at least 256 bytes long, and if smb decides to use 4 sets of 256 packs of wavetables to simulate complex sound like C64 filtered waves or PWM VGM file will anyway be huge, but in the NuVGM we can store each wave only once, even if there are 1024 or more of them).

A special block where chip outputs can be assigned to output channels (kinda like Furnace's Patchbay). This is for chips that have more than two output channels (ES5503/5505/5506, OPL3, etc.). The player itself supports up to 256 audio channels, `0xff` being the NULL channel.

## The format MUST support all the chips that are even vaguely sound chips
This includes MOS Technology SID (both models, with customizable filter curve!), NES, etc. The format should not care if there is any other widespread existing format for the chip. If it chip-tunes, it gets added.

# Preliminary spec

All multibyte variables are stored little-endian, so `0x12345678` is written in file as `0x78 0x56 0x34 0x12`.

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| `nVGM` format magic | ASCII string | 4 |  |
| EOF pointer | `uint64_t` | 8 | `(file_size - 8)` |
| Version | `uint64_t` | 8 | version |
| GD3 offset  | `uint64_t` | 8 | relative; things like music engine Hz rate, song/album cover, author, jump table for playlist if all the OST is stored inside one file etc. can be stored there idk |
| Initial sample rate | `uint64_t` | 8 | may change throughout the file |
| № of samples | `uint64_t` | 8 | total number of samples (sum of all wait commands) |
| NuVGM data offset | `uint64_t` | 8 | realtive offset to the start of actual data (?? maybe unnecessary?) |

Blocks of data follow.

Block structure:
| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Block name | ASCII string  | 16 |  |
| Block version | `uint64_t` | 8 |  |
| Block size | `uint64_t` | 8 | size of the block (excluding this field, block name and block version, so `(block_size - 32)`) |
| Block data | binary data | Block size | whatever inside the block |

Player must skip unknown blocks.

## Defined blocks

Here the block data of main blocks is described.

### Header

Block name is `NuVGM fileheader`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Chip list size | `uint32_t`  | 4 |  |
| Chip list | `uint16_t` array  | 2 * Chip list size | Each chip type has 16-bit signature. There can be unlimited amount of chips of the same type, but settings are individual per chip. |

Then the array of chip settings follows, one per chip:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Volume | `float` | 4 |  |
| Panning | `uint16_t` | 2 | `0x0000` = left, `0x8000` and `0x7fff` = center, `0xffff` = right |
| Clock | `uint32_t` | 4 |  |
| Version of additional parameters section | `uint32_t` | 4 |  |
| Size of additional parameters section | `uint32_t` | 4 |  |
| Additional parameters section | binary data | Size of additional parameters section |  |

The section can contain whatever chip needs, like output routing, filter curve and chip sub-model for MOS Technology SID etc. You can also store extended panning data here.
Special opcode is provided to change some chip settings mid-song, so this field can contain an array of chip's configurations/settings (here settings that can't be changed by register writes are meant, like clock speed, final mixing volume etc.).

### RAM writes

Block name is `NuVGM RAM writes`

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

The index of the entry in this array is used in data stream to reference it, so player must build up some kind of LUT when scanning the file to make it possible.
Special values of data/compression type may be reserved to signal that the range of RAM must be filled with 0's or a particular byte/repeating byte sequence. In this case the actual data may be omitted.

### ROM images (or parts of them; includes banks and whatever)

Block name is `NuVGM ROM data  `

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

The index of the entry in this array is used in data stream to reference it, so player must build up some kind of LUT when scanning the file to make it possible to assign different ROMs to different chips.
Special values of data/compression type may be reserved to signal that the range of ROM must be filled with 0's or a particular byte/repeating byte sequence. In this case the actual data may be omitted.

### Loop points

Block name is `NuVGM looppoints`

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Number of loop points | `uint32_t` | 4 |  |

Then the array of sections follows:

| Field name | Field type | Field size (in bytes) | Comments |
| ------------- | ------------- | ------------- | ------------- |
| Loop begin absolute offset | `uint64_t` | 8 |  |
| Loop end absolute offset | `uint64_t` | 8 |  |

A special opcode is used for halting the playback (see below).

### Main logged data block

Block name is `NuVGM loggeddata`

The block contains data in form of opcodes and their operands stream, much like the VGM.

## Opcodes

In the main logged data block there is a stream of commands. Each command has an opcode as its first byte, then one or more bytes of data follow. Opcodes are described there:
