# SF2 Extended Limits Proposal

This proposal massively extends the allowed number of generators, modulators and zones in an SF2 file, also extending names.

**Note:** This proposal assumes that the reader is familiar with the SoundFont2 specification, version 2.04.

## The problem
Currently, the indexes to zones, generators and modulators are limited to 65,536 each.

This is the biggest problem with generators, especially for large SF2 files (such as [this one](https://musical-artifacts.com/artifacts/2525)),
and [DLS to SF2 conversions](https://github.com/spessasus/spessasynth_core/wiki/DLS-Conversion-Problem).

## The Solution

### Goals

The solution must meet the following criteria:

- It can't violate the SoundFont2 specification (no redefining the SF2 structure)
- It must be straightforward to implement (i.e., no new complex chunks)
- It can't alter the synthesis model
- The limits must be extended significantly

### Proposed Solution

Proposed solution involves adding a new `LIST` chunk into the `INFO-list`: `xdta-list`. (`xdta` stands for extended data)

`xdta-list` contains all the `pdta-list` chunks, in the same order`. That is:
- `phdr`
- `pbag`
- `pmod`
- `pgen`
- `inst`
- `ibag`
- `imod`
- `igen`
- `shdr`

All these chunks have **exactly the same structure and rules** as their counterparts in the `pdta-list`.
This is intended to allow software implementations to reuse their `pdta-list` parser.

Software supporting this chunk is recommended to write this chunk only when any value or name exceeds the standard SF2 limits.
It is allowed for the supporting software to ignore the name limit extensions if 40 characters cannot be displayed.

These chunks are combined with their `pdta-list` counterparts.
The exact way of combining each of the SF2 values is described below.

### xdta chunks
#### phdr
- CHAR achPresetName[20] - the second half of the preset name, extending the limit to 40 characters. To get the full name, combine these, like this: `name = pdtaName + xdtaName;`
- WORD wPreset - unused, left at zero.
- WORD wBank - unused, left at zero.
- WORD wPresetBagNdx - the higher two bytes of the number, raising the index limit to 4,294,967,296. Like this: `index = (xdtaWord << 16) | pdtaWord;`
- dwLibrary, dwGenre, dwMorphology - all unused like in the original specification.

#### pbag
- WORD wGenNdx - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`
- WORD wModNdx - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`

### pmod
This chunk is only intended to allow reusing the `pdta-list` parser. It only contains the terminal modulator record.

### pgen
This chunk is only intended to allow reusing the `pdta-list` parser. It only contains the terminal generator record.

#### inst
- CHAR achInstName[20] - the second half of the instrument name, extending the limit to 40 characters. To get the full name, combine these, like this: `name = pdtaName + xdtaName;`
- WORD wInstBagNdx - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`

#### ibag
- WORD wGenNdx - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`
- WORD wModNdx - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`

### imod
This chunk is only intended to allow reusing the `pdta` parser. It only contains the terminal modulator record.

### igen
This chunk is only intended to allow reusing the `pdta` parser. It only contains the terminal generator record.

### shdr
- CHAR achSampleName[20] - the second half of the sample name, extending the limit to 40 characters. To get the full name, combine these, like this: `name = pdtaName + xdtaName;`
- DWORD dwStart - unused, left at zero (64-bit numbers are not viable in the RIFF structure)
- DWORD dwEnd - unused, left at zero.
- DWORD dwStartloop - unused, left at zero.
- DWORD dwEndloop - unused, left at zero.
- DWORD dwSampleRate - unused, left at zero.
- BYTE byOriginalPitch - unused, left at zero.
- CHAR chPitchCorrection - unused, left at zero.
- WORD wSampleLink - the higher two bytes of the number: `index = (xdtaWord << 16) | pdtaWord;`
- SFSampleLink sfSampleType - unused, left at zero. 

## Rationale

- All index limits have been raised to 4,294,967,296. This ensures that even complex SF2 files (ones that use a lot of zones/samples yet fit in the 4GB limit) can freely be expanded without having to care about the 16-bit limit.
- Extended name limits allow full instrument names. For example: `*Detuned EP 2` can be turned into `*Detuned Electric Piano 2`.
- Fully backwards compatible with the existing SF2 parsers, even allowing them to play a limited part of the extended SF2 file.
- `pdta-list` parsers can be reused, reducing the complexity of implementation.