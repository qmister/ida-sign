              FORMAT OF A PATTERN FILE USED BY IDA FLAIR
              ==========================================

What is a pattern file
----------------------

A PAT contains information about object modules from a library.
Usually this file is generated by PLB or PCF utilities.
PLB stands for "parse library" and processes OMF object libraries.
PCF stands for "parse COFF" and processes AR object libraries.
A collection of PAT files is processed by the "sigmake" utility
which produces a signature file. So, a normal flow of data is:


          PLB or PCF                     Sigmake
Library -------------->  PatternFile  ------------>  SignatureFile

Sigmake can take one or more pattern files and produce one signature file.
If you want to make signature files for a library you have,
you may take PLB or PCF and try to generate a pattern file.
But if your libraries are not in OMF or AR format, they will fail.
In this case you need to write your own preprocessor of libraries.

How you write this preproccesor, what programming language you use,
on what platform, etc - is not important. The only requirement for the
preprocessor is to produce a correct PAT file.
Below is a detailed description of the format.

Format of PAT file
------------------

A PAT file is a text file.
Each object module from a library is represented as a separate line.
Length of a line is not limited.
Let's look at an example (the first line is an example,
the second is a ruler to make explanations):

558BEC8B5E04D1E3F787....02007406B8050050EB141EB43F8B5E048B4E0AC5 0B B56E 002F :0000 __read ^000B __openfd ^002C __IOERROR ....5DC3
pppppppppppppppppppppppppppppppppppppppppppppppppppppppppppppppp ll ssss LLLL gggggggggggg rrrrrrrrrrrrrrrrrrrrrrrrrrrrrr tttttttt

This line describes one module from a library. The module starts with a
sequence of bytes 558BEC8B5E04D1E3F787, then there are 2 variable bytes,
then bytes 02007406B8050050EB141EB43F8B5E048B4E0AC5.
If we calculate CRC16 on the following 0B bytes, it will be equal to B56E.
The length of the module is 002F bytes. The module defines one global name
"__read", it is located at the start of the module (offset 0000).
Also the module refers to two names: __openfd (from offset 000B)
and __IOERROR (from offset 002C).
All the remaining bytes of the module are written at the end of the line
(this is why a line might be veery long; however, in this particular case
the module is short): ....5DC3

Format of each line:
        p - PATTERN BYTES (64 positions)
            space
        l - 2 positions contain ALEN (example:12)
            space
        s - 4 positions contain ASUM (example:1234)
            space
        L - 4 positions contain TOTAL LENGTH OF MODULE IN BYTES (example:1234)
            space
        g - LIST OF PUBLIC NAMES
        r - LIST OF REFERENCED NAMES
        t - TAIL BYTES

where

PATTERN BYTES:
     first 64 characters represent first 32 bytes of module.
     If value of a byte is variable, it is represented as ".."
     Otherwise a byte is represented by 2 hexadecimal digits (XX)

ALEN is length of block starting at 32th byte of the module
     used to calculate CRC16. This block can't contain variable
     bytes. Maximal length of this block is 255 bytes.

ASUM is CRC16 of the aforementioned block.

TOTAL LENGTH OF MODULE IN BYTES - contains what it says
     Total length of a module should be < 0x8000.

LIST OF PUBLIC NAMES:
     Each public name is represented as
       :XXXX name
     where XXXX is offset of the name from the module start.
     There must be at least one public name. If a module has no
     public names, parselib should create a name ":0000 ?"
     If the offest is negative, it is represented like this:
       :-XXXX name
     If a name is local, it is represented as
       :XXXX@ name
     i.e. there is '@' after the offset.
     Elements of this list are separated by spaces.

LIST OF REFERENCED NAMES:
     Each referenced name is represented as
       ^XXXX name
     where XXXX is offset of the location refering to the name.
     Obviously, bytes at this location are variable.
     Special for 80x86 processors: some linkers convert far calls
     to near calls in 16bit segments:

     	0000: 9A........	call	far ptr xxx

     is converted to

        0000: 90		nop
     	0001: 0E		push	cs
	0002: E8....		call	near ptr xxx

     Therefore, parselib should mark byte 9A as variable and
     set location offset of the fixup to 0002, not 0001.

TAIL BYTES:
     Have the same format as the first 32 bytes.
     Tail of the module starts at the end of the CRC16 block.

All numbers in a PAT file are hexadecimal.
A PAT file should be ended with a special line with 3 minus signs:
---

Limitations:
     Total length of a module should be < 0x8000.
     Too short modules (less than 4 constant bytes) should not be
     included in the PAT file. However, if a module have a referenced
     name, it can be included in the PAT file.

Examples:

558BEC8B4604C706....0000A3....5DC38B0E....8B1E....BA5A01B8354EE8 00 0000 0037 :0000 _srand :0011 _rand ^0021 N_LXMUL@ ....05010083D2008916....A3....A1....9925FF7FC3
558BEC8B5E04D1E3F787....02007406B8050050EB141EB43F8B5E048B4E0AC5 0B B56E 002F :0000 __read ^000B __openfd ^002C __IOERROR ....5DC3

1111111111111111111111111111111111111111111111111111111111111111 22 3333 4444 555555555555				  66666666

1 - pattern bytes
2 - ALEN
3 - ASUM
4 - MODLEN
5 - PUBLIC NAME
6 - TAIL BYTES

__read 		 refers to	 __openfd and __IOERROR
_srand and _rand refer  to	 N_LXMUL@

============================================================================
