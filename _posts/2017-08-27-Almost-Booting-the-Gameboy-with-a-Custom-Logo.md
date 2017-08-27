---
title:  "(Almost) Booting the Gameboy with a Custom Logo"
date:   2017-08-27
---
In 2003 neviksti managed to extract the original Gameboy boot ROM by [literally putting the CPU under a microscope](http://www.neviksti.com/DMG/). The ROM on the chip was soon decoded, revealing the [bootstrap program](http://gbdev.gg8.se/wiki/articles/Gameboy_Bootstrap_ROM) responsible for reading and parsing the header of the game cartridge. The program is pretty simple; it reads the header stored on the cartridge, validates it, scrolls the Nintendo logo and plays the di-ding sound. If the header is valid it then starts the program at the entry point. Interestingly, a side-effect of this process allows anyone with a hex editor and too much time on their hands to change the appearance of the Nintendo logo.

## The Header

This is the header from a game cartridge, starting at offset 0100:

```
0100 : 00 C3 50 01 CE ED 66 66 CC 0D 00 0B 03 73 00 83
0110 : 00 0C 00 0D 00 08 11 1F 88 89 00 0E DC CC 6E E6
0120 : DD DD D9 99 BB BB 67 63 6E 0E EC CC DD DC 99 9F
0130 : BB B9 33 3E 53 55 50 45 52 20 4D 41 52 49 4F 4C
0140 : 41 4E 44 00 00 00 00 01 01 00 00 01 01 9D 5E CF
```

- 0100-0103 is the entry point for the program stored on the cartridge. This is almost always `00 C3 50 01`, which translates to a `NOP` followed by a `JP 0150h` (Where the address is stored as LL HH, `50 01`).

- 0104-0133 contains a "secret" validation code.

- 0134-014C contains information about the cartridge and the program on it (for instance the title is located at offset 0134-0143).

- 014D contains an 8-bit checksum of the header bytes at 0134-014C. This checksum is validated by the bootloader.

- 014E-014F is a 16-bit checksum of the entire ROM. This checksum is not validated by the bootloader.

  The main program the starts at offset 0150.

More information about the header can be found [here](http://gbdev.gg8.se/wiki/articles/The_Cartridge_Header)

## The Validation Code

Let's look at the validation code at 0104-0133:

```
0104 : CE ED 66 66 CC 0D 00 0B 03 73 00 83 00 0C 00 0D
0114 : 00 08 11 1F 88 89 00 0E DC CC 6E E6 DD DD D9 99
0124 : BB BB 67 63 6E 0E EC CC DD DC 99 9F BB B9 33 3E
```

This code is the same for all Gameboy games and has to be present or the bootloader will hang. However, the bootloader will run all the way to the di-ding even if everything is invalid, so we can create a ROM filled with all bytes set to `00`  and it will still do *something*. We need at least 336 (014F) bytes or the ROM won't boot at all, since part of the header would be missing.

Here's our rom:

```
0000 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
...  : ... 
0100 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0110 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0120 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0130 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0140 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

And here is what it looks like in BGB:

![DMG-all-00]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-all-00.gif)

Where did the logo go? It has to be affected by the validation code, right? What happens if we change the code to all `FF`?

Here's our new ROM:

```
0000 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
...  : ... 
0100 : 00 00 00 00 FF FF FF FF FF FF FF FF FF FF FF FF
0110 : FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF
0120 : FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF
0130 : FF FF FF FF 00 00 00 00 00 00 00 00 00 00 00 00
0140 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

And here it is in BGB:

![DMG-all-FF]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-all-FF.gif)

The logo isn't just affected by the validation code; it *is* the logo (surprise!). Now we just need to figure out how to decode it.	

# Decoding the Logo

The logo is 48x8 pixels and monochrome. Each "pixel" is actually a block of four dots on the matrix. The copyright logo is drawn separately and cannot be altered. The area inside the red markings is our canvas.

![DMG-Logo]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-Logo.png)

Our code 48 bytes long giving us, you guessed it, 48x8 bits in total. Each bit should correspond to a pixel. Let's assume pixels are added left to right, top to bottom. Think of our 48-byte code as a 48x8 bit matrix. By setting the most significant bit to 1 the top right pixel should come on.

![DMG-firstbit]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-firstbit.png)

Fantastic! Our assumption keeps working until we flip the fifth bit. Then this happens:

![DMG-fifthbit]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-fifthbit.png)

Each nibble corresponds to a row of 4 pixels added top to bottom. But when we reach the third byte this happens.

![DMG-17thbit]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-17thbit.png)

The first nibble of the third byte is added to the right of the first block. This pattern repeats for the entire top half of the logo. The lower half is then added in the same way. The nibbles are mapped to the logo in the following way:

     0   4   8  12  16  20  24  28  32  36  40  44
     1   5   9  13  17  21  25  29  33  37  41  45
     2   6  10  14  18  22  26  30  34  38  42  46
     3   7  11  15  19  23  27  31  35  39  43  47
    48  52  56  60  64  68  72  76  80  84  88  92
    49  53  57  61  65  69  73  77  81  85  89  93
    50  54  58  62  66  70  74  78  82  86  90  94
    51  55  59  63  67  71  75  79  83  87  91  95


Armed with this knowledge and a healthy dose of python we should be able to extract the Nintendo logo from the header.

```python
from itertools import chain
from PIL import Image

bytes_raw = bytes.fromhex(
    'ce ed 66 66 cc 0d 00 0b 03 73 00 83' \
    '00 0c 00 0d 00 08 11 1f 88 89 00 0e' \
    'dc cc 6e e6 dd dd d9 99 bb bb 67 63' \
    '6e 0e ec cc dd dc 99 9f bb b9 33 3e'
    )

# Split bytes into separate bytes for upper and lower nibbles
logo_nibs = b''.join(
    bytes([b >> 4, b & 15]) for b in bytes_raw
    )

# The mapping can be generated as a chain of ranges
mapping = chain.from_iterable(
        range(x+y, 48+y, 4) for y in (0, 48) for x in range(4)
    )

# Sort the nibbles according to the mapping
sorted_nibs = [logo_nibs[x] for x in mapping]

# Recombine the nibbles into a 48-byte string
logo_out = bytes(
    (sorted_nibs[n] << 4) | sorted_nibs[n+1] for n in range(0, 96, 2)
    )

Image.frombytes('1', (48, 8), logo_out).save('logo.bmp')
```

This could definitely be shorter, but I took some extra steps to make it readable. Here's the output:

![DMG-Logo-Decoded]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-Logo-Decoded.png)

It works!

# Encoding a Logo

Now that we can decode a logo, encoding our own logo should just be a matter doing the same process in reverse. This is the logo I want to encode:

![DMG-mylogo]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-mylogo.png)

We can reuse most the code from our decoding program. All we need to do is to reverse the sorting. This can be achieved by sorting based on the index of each nibble in the mapping. I eneded up with this code:

```python
from itertools import chain
from PIL import Image

logo_raw = Image.open('mylogo.bmp').tobytes()

logo_nibbs = b''.join(
    bytes([b >> 4, b & 15]) for b in logo_raw
    )

mapping = chain.from_iterable(
    range(x+y, 48+y, 4) for y in (0, 48) for x in range(4)
    )

# The mapping has to be converted to a list since a chain doesn't have an index
mapping = list(mapping)

# The sorting is reversed by looking at the index of each nibble in the mapping
sorted_nibbs = [logo_nibbs[mapping.index(x)] for x in range(96)]

logo_bytes = bytes(
    (sorted_nibbs[n] << 4) | sorted_nibbs[n+1] for n in range(0, 96, 2)
    )

# The logo bytes are padded with zeroes to fill out the header
bytes_out = b'\x00'*260 + logo_bytes + b'\x00'*28

with open('mylogo.gb', 'wb') as f:
    f.write(bytes_out)
```

Once again, this could definitely be shorter. You could probably do it in a single list comprehension if readability wasn't an issue. Anyway, here's the result in BGB:

![DMG-dodslaser]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-dodslaser.gif)

BGB will complain that pretty much everything is broken in this ROM, and that it will not play on a real gameboy. Most importantly, it will say that the logo verification will fail. While this is true, the logo will still be scrolled on real hardware. This works because the logo is actually read twice by the bootloader. Once to be scrolled, and once again to be validated. Some pirate gamecarts abuse this fact by replacing the logo in the header after it is read the first time, making a custom logo scroll while the header still passes validation. I won't get into that right now because I don't have the hardware (or skills) to do it (hence the "almost").

For now I'm happy with my custom logo being scrolled on a Gameboy.

![DMG-hw]({{ site.url }}\assets\images\posts\2017-08-27-Almost-Booting-the-Gameboy-with-a-Custom-Logo\DMG-hw.gif)