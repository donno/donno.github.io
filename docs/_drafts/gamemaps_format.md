GameMaps Format
===============

The GamesMap format was used in a number of games made by id Software around
1991. The two most well known and popular games games that used this format
were Wolfenstein 3D and Commander Keen 4 (as well as 5 and 6).

I started with Wolfenstein 3D (Wolf3D) then took a break from that to try to
open Commander Keen 4 levels.

## Wolfenstein 3D 

The extension identifies if it is from the full version of the game or the
shareware (these days known as demo or trial) version. The extension WL6 is
used for files of the full game and the extension WL1 is used for files of the shareware version.

### MAPHEAD
The map header file.

|Property      |Type       |Description                                       |
|--------------|-----------|--------------------------------------------------|
|Magic Number  | uint16le  | Magic number that identifies the file format. If this is 0xABCD it means there will be RLEW compression. |
|Level Pointers| int32le   | The pointers/offset to the levels. A value of -1 implies there is no level there.|

For Wolfenstein 3D, this is a separate file, however for Commander Keen this
file is within the executable itself.

### VSSWAP.WL6
This contains the wall tiles.

```c
    struct Header
    {
        uint16_t ChunkCount;
        uint16_t SpriteStart;
        uint16_t SoundStart;
        uint32_t ChunkOffsets[ChunkCount];
        uint16_t ChunkLengths[ChunkCount];
    };
```

The SpriteStart and SoundStart are the indices in the chunks.
The chunks are arranged in the following order:
* Walls
* Sprites
* Sounds

**TODO**: Cover how to decode thw wall textures
**TODO**: Fininsh writing up about this.

## Commander Keen 4

### MAPHEAD
Unlike Wolfenstein 3D which has this in a separate file, Commander Keen this is
found within executable itself.

To make matters harder for us the executable itself is compressed with
LZEXE v0.91. I haven't looked into decompressing it myself so instead I found
success using [unlzexe][0].

Firstly, I check the executable given starts with the two magic bytes of MZ to
indicate it is infact a DOS MZ file. Next, seek to the relocation table offset
which is at 24-bytes from the start of the file. The relocation table offset is
2-bit unsigned little endian integer, which I expect to be 0x1C.

From there seek to the relation table offset (0x1C) and read four bytes,
if those for bites are LZ91 it means the executable is compressed and I
recommend decompressing it with unlzexe unless you wish to write that part.

Next jump to the map header at 0x27630, from here, I used the same function
I wrote for the Wolfenstein 3D. and it handled it.

## gamemaps.ck4

The first eight bytes are the magic number that identifies this as the expected
file format. In this case it will be `TED5v1.0`.

For each of the level offsets read from the header, in the file we seek to the
location and start reading the level data. This is the same as Wolf3D.

[0]: http://www.shikadi.net/keenwiki/UNLZEXE
