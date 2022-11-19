# __**1. Header `[..0x19]`**__  
• The first two bytes represent a short containing the chunk's version in hex. So far only `12 = Aquatic` is known.
• The next 8 bytes represent two integers with the chunk's X and Z position respectively.  
• The last 16 bytes are two longs containing the `LastUpdate` and `InhabitedTime` tags.  
  
```diff
   |00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
   |-----------------------------------------------
+00|00 0C 00 00 00 01 00 00 00 05 00 00 00 00 00 00
   |^^^^^ ^^^^^^^^^^^ ^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^
   |Ver.  Chunk X-Pos Chunk Z-Pos LastUpdate...
+10|00 00 00 00 00 00 00 00 00 00 ?? ?? ?? ?? ?? ??
   |^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^
   |...   InhabitedTime
```  
# __**2. Section Header `[0x1A..0x4B]`**__  
> **\*** "Sections" are vertically-aligned cubes of 16×16×16 blocks (4096), and there are 16 of them in each chunk. Starting from 0 (y0) up to 15 (y255)  
  
• The first 0x2 `[0x1A..0x1B]` bytes define how large the block data is (multiply by `0x100` to get the actual size)  
• The next 0x20 bytes `[0x1C..0x3B]` contain 16 shorts defining offsets of where sections start called the `jump table`. If an offset matches the block data length (shown above), then there should be no more sections to parse.  
• The next 0x10 bytes `[0x3C..0x4B]` are the sizes of sections. Multiply by `0x100` to get the actual size. If a size is `0`, then there's probably no more sections to parse. (It is recommended to use the `jump table` if the whole chunk is loaded into memory for each section and not bother with sections sizes and reading into smaller chunks)
# __**3. Section Data `[0x4C..(*X+0x4C)]`**__
> **\*** X is defined in the Section Header as the "block data size" (The total length of the all the block data sections) `[0x1A..0x1B]`
> **†** Y is defined in the Section Header as the "section offset". Add `0x4C` to get the actual address in the chunk data. `[0x1C..0x3B `  
> **‡** Z is defined in the Section Header as the "section size". `[0x3C..0x4B]`  
> **⁂** Grids are cubes of 4×4×4 blocks (64 blocks), and there are 64 of them per section (64x64 = 4096 blocks per section). They are stored in a 4x4x4 grid of the section. 
> **⸸** The Block ID is the first 12 bits and the Block Data is the last 4 bits.  
> **Code:** `Byte1 Byte2` - `Block ID = ((byte2 << 4) + ((byte1 & 0xF0) >> 4)) & 0x1FF;` + `Data Value: byte1 & 0xf;`  
> **Example:** `0x70 0x05` - `Block ID: 0x57 (87)` + `Data Value: 0x0 (0)`

For each section:  
• The first 0x80 bytes `[†Y..Y+0x7F]` is the grid`⁂` index table. Grid indicies are stored in YZX format. `gridIndex = Y† + ((gx * 16) + (gz * 4) + gy)`  
• The remaining bytes `[Y+0x80..(Y+(Z-0x80))]` define the 'palette' data, storing what blocks are used in the section.  
• Each "grid index" consist of 2 bytes. The first nybble of the second byte defines the grid's format, which defines how the blocks are stored and how many bits are stored. Nybbles `3`, `0`, and `1` build the offset of where the grid's palette is stored in that section data. `offset = (nybble3 << 8 | nybble0 << 4 | nybble1) * 4` / `format = nybble2`  
• **Example:** `0x9C 0x40` - `Format = 0x4` + `Offset = 0x270`

• How to parse blocks from the format:

| Format | Palette Size | Description |
| :---: | :---: | --- |
| 0 | 1 (0x01) | The entire grid is filled with just one type of block. Block is stored in the grid index itself (`byte 0` and `byte 1` make up the block data⸸). Copy `byte 0` and `byte 1` into the block data array 128 times (64 times each and alternating) |
| E | 128 (0x80) | The palette for this grid is is 0x80 bytes long and are read straight into the block data array without parsing. |
| F | 256 (0x100) | The format is the same as above and there is an extra 0x80 bytes at the end that are read straight into "Liquid" data without parsing. |
| 2 | 12 (0x0C) | The grid contains only 2 block types. The first 2 shorts of the palette are the blocks. The last 8 bytes define bits that store the block's positions. (1 bit per block) |
| 3 | 20 (0x14) | The format is the same as above and there is an extra 8 bytes at the end that point to the grid and are read into "Liquid" data. (1 bit per block) |
| 4 | 24 (0x18) | The grid contains up to 4 block types. The first 4 shorts of the palette are the blocks. The last 16 bytes define bits that store the block's positions. (2 bits per block) |
| 5 | 40 (0x28) | The format is the same as above and there is an extra 16 bytes at the end that point to the grid and are read into "Liquid" data. (2 bits per block) |
| 6 | 40 (0x28) | The grid contains up to 8 block types. The first 8 shorts of the palette are the blocks. The last 28 bytes define bits that store the block's positions. (3 bits per block) |
| 7 | 64 (0x40) | The format is the same as above and there is an extra 28 bytes at the end that point to the grid and are read into "Liquid" data. (3 bits per block) |
| 8 | 64 (0x40) | The grid contains up to 16 block types. The first 16 shorts of the palette are the blocks. The last 32 bytes define bits that store the block's positions. (4 bits per block) |
| 9 | 96 (0x60) | The format is the same as above and there is an extra 32 bytes at the end that point to the grid and are read into "Liquid" data. (4 bits per block) |

❖ If there are less blocks in the grid than the limit then it the rest will be filled with `0xFF`.

❖ The extra pointers that gets parsed into "Liquid" data work the same way as the pointers to the blocks.

❖ The code below shows how to get the block positions from the data (the previous explanation was incorrect):  

Here is working code for parsing these sub chunks (without extra "Liquid" data):
```
template <size_t BitsPerBlock>
    bool Parse(uint8_t const* buffer, uint16_t* grid) {
        int size = (1 << BitsPerBlock) * 2;
        std::vector<uint16_t> palette(size);
        std::copy_n(buffer, size, palette.begin());
        for (int i = 0; i < 8; i++) {
            uint8_t v[BitsPerBlock];
            for (int j = 0; j < BitsPerBlock; j++) {
                v[j] = buffer[size + i + j * 8];
            }
            for (int j = 0; j < 8; j++) {
                uint8_t mask = (uint8_t)0x80 >> j;
                uint16_t idx = 0;
                for (int k = 0; k < BitsPerBlock; k++) {
                    idx |= ((v[k] & mask) >> (7 - j)) << k;
                }
                if (idx >= size) [[unlikely]] {
                    return false;
                }
                int gridIndex = (i * 8 + j) * 2;
                int paletteIndex = idx * 2;
                grid[gridIndex] = palette[paletteIndex];
                grid[gridIndex + 1] = palette[paletteIndex + 1];
            }
        }
        return true;
    }
```

Here is working code for parsing these sub chunks (with extra "Liquid" data):
```
template <size_t BitsPerBlock>
    bool ParseWithLayers(uint8_t const* buffer, uint16_t* grid, uint16_t* submergedGrid) {
        int size = (1 << BitsPerBlock) * 2;
        std::vector<uint16_t> palette(size);
        std::copy_n(buffer, size, palette.begin());
        for (int i = 0; i < 8; i++) {
            uint8_t v[BitsPerBlock];
            uint8_t vSubmerged[BitsPerBlock];
            for (int j = 0; j < BitsPerBlock; j++) {
                int offset = size + i + j * 8;
                v[j] = buffer[offset];
                vSubmerged[j] = buffer[offset + BitsPerBlock * 8];
            }
            for (int j = 0; j < 8; j++) {
                uint8_t mask = (uint8_t)0x80 >> j;
                uint16_t idx = 0;
                uint16_t idxSubmerged = 0;
                for (int k = 0; k < BitsPerBlock; k++) {
                    idx |= ((v[k] & mask) >> (7 - j)) << k;
                    idxSubmerged |= ((vSubmerged[k] & mask) >> (7 - j)) << k;
                }
                if (idx >= size || idxSubmerged >= size) [[unlikely]] {
                    return false;
                }
                int gridIndex = (i * 8 + j) * 2;
                int paletteIndex = idx * 2;
                int paletteIndexSubmerged = idxSubmerged * 2;
                grid[gridIndex] = palette[paletteIndex];
                grid[gridIndex + 1] = palette[paletteIndex + 1];
                submergedGrid[gridIndex] = palette[paletteIndexSubmerged];
                submergedGrid[gridIndex + 1] = palette[paletteIndexSubmerged + 1];
            }
        }
        return true;
    }
```

# __**4. Block Light & Sky Light `[X..??]`**__  
> **\*** X is the total length of the all the block data sections, as shown in the **Section Data**. Add `0x4C` to get the starting offset. `[0x1A..0x1B]`

❖ Skylight and Blocklight are nybble arrays of 0x8000 bytes stored in XZY format

❖ There are 4 different "sections" of light, the first 2 are the SkyLight, and the last 2 being the BlockLight.

| For each section |
| :---: |
| The first 4 bytes (*int) times by 128 plus 128 defines the length of that section `(*int + 1) * 0x80` or `*int * 0x80 + 0x80` |
| The first 0x80 bytes is the header for the data |


| For each byte in the header |
| :---: |
| If the byte is `0x80` then fill 128 bytes of Skylight/Blocklight with `0x00` |
| If the byte is `0x81` then fill 128 bytes of Skylight/Blocklight with `0xFF` |
| If the byte is not `0x80` or `0x81`, then read 128 bytes of data with offset of `byte value * 0x80 + 0x80` from the section into Skylight/Blocklight data. (Note that the `+0x80` is there because the header is 128 bytes long) |

# __**5. Height Map, TerrainPopulatedFlags, & Biomes `[0x??..(0x??+0x202)]`**__  
• The first 0x100 bytes store the HeightMap byte array.  
• The next 0x2 bytes store the short `TerrainPopulatedFlags`.  
• The last 0x100 bytes store the Biomes byte array.  
# __**6. Raw NBT Data `[0x??..]`**__  
• The rest of the file contains raw NBT data after all the other data.
