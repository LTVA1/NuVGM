# NuVGM
A humble attempt to fix the issues of VGM file format...
# Original ideas by tildearrow
my proposal for a successor to VGM (i'd call it NuVGM (thanks Iyatemu for the name idea), but feel free to use any name):
- a "list of chips". every chip has a few fields: type, clock rate, volume, panning, parameter length and parameters (parameters which are not recognized by a player which only supports an old version of NuVGM are ignored)
--- the type is 16-bit to ensure we can support up to 65536 chip types (should be more than enough)
--- the volume is 32-bit floating point
- a "logging rate" field
- an UTF-8-like chip selection scheme, when there are more than 128 chips:
--- e.g. if the "write to 8/8 register" command is 10, then we would do: `10 xx... yy zz` (where xx is the chip index, yy is the address and zz is the data)
--- if we want to access chip 0: `10 00 yy zz`
--- if we want to access chip 128: `10 C2 80 yy zz`
--- if we want to access chip 66376: `10 F0 90 8D 88 yy zz`
--- this way the format can store songs which use up to ~~2147483647~~ 1114111 (`0x10FFFF`) chips
- in order to remain compact, a few dedicated commands to write to the first four or so chips will be added
- different register write commands for different register widths
--- in the format addr/data: 8/8, 16/8, 16/16 (e.g. QSound), 32/8, 32/16, 32/32
- data blocks are chip-independent
- an "assign ROM data" command to bind a ROM to a chip, like so: `50 xx yyyy zz` (where xx is the chip index, yyyy is the data block and zz is the ROM type (for chips which can be assigned more than one, e.g. YM2610))
--- this also allows for some degree of bank switching (to be discussed further)
- DAC streams can use any data block, write to any address and allow for loop points
- an extensible, backward and forward compatible tag format like ID3v2 (should we call it GD3v2?)
--- tag data is UTF-8 (enough of Windows-specific UTF-16)

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

DAC streams/chip RAM updates/special registers updates (like wavetables on Gameboy/N163) are to be isolated in special blocks and given a unique 16 or 32-bit identifier, and an opcode can reference those blocks by that index. This can be done in order for optimization program to be able to handle lots of different writes, storing each write only one (e.g. wavetable update on ES5503 is at least 256 bytes long, and if smb decides to use 4 sets of 256 packs of wavetables to simulate complex sound like C64 filtered waves or PWM VGM file will anyway be huge, but in the NuVGM we can store each wave only once, even if there are 1024 or more of them).
