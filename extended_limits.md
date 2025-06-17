# SF2 Extended Limits Proposal

This proposal significantly extends the allowed number of generators, modulators, and zones in an SF2 file. It also increases the maximum length for names.

**Note:** This proposal assumes that the reader is familiar with the SoundFont 2 specification, version 2.04.

---

## The Problem

Currently, the indexes for zones, generators, and modulators are limited to 65,536 (16-bit maximum unsigned number).

This limitation becomes problematic for large SF2 files, such as [this one](https://musical-artifacts.com/artifacts/2525), 
and for [DLS to SF2 conversions](https://github.com/spessasus/spessasynth_core/wiki/DLS-Conversion-Problem), 
where a high number of generators or zones is often required.

## The Solution

### Goals

The solution must meet the following criteria:

- It must not violate the SoundFont 2 specification (i.e., no redefinition of the SF2 structure)
- It must be straightforward to implement (no new complex chunk types)
- It must preserve the existing synthesis model
- It must extend current limits significantly

### Proposed Approach

The proposed solution introduces a new `LIST` chunk within the `INFO` list, named `xdta-list`. (`xdta` stands for *extended data*.)

This chunk replicates all of the `pdta-list` chunks, in the same order:

- `phdr`
- `pbag`
- `pmod`
- `pgen`
- `inst`
- `ibag`
- `imod`
- `igen`
- `shdr`

All of these chunks follow **exactly the same structure and rules** as their `pdta-list` counterparts. 
This design allows software implementations to reuse their existing `pdta` parsers with minimal changes.

Supporting software should only write the `xdta-list` if a name or index exceeds standard SF2 limits. 
If it cannot determine the need for writing the `xdta-list`, it should write the chunk anyway.

If an application cannot display more than 40 characters, it may ignore the extended name fields.

Each `xdta` chunk combines with its `pdta` counterpart to form a 32-bit value. The exact behavior for each chunk is described below.

---

## `xdta` Chunk Descriptions

### `phdr`
- `CHAR achPresetName[20]` — second half of the preset name (combined with `pdta` name for up to 40 characters).
- `WORD wPreset` — unused, set to zero.
- `WORD wBank` — unused, set to zero.
- `WORD wPresetBagNdx` — upper 16 bits of the preset bag index:  
  `fullIndex = (xdtaWord << 16) | pdtaWord`
- `dwLibrary`, `dwGenre`, `dwMorphology` — unused, set to zero.

### `pbag`
- `WORD wGenNdx` — upper 16 bits of the generator index:  
  `fullIndex = (xdtaWord << 16) | pdtaWord`
- `WORD wModNdx` — upper 16 bits of the modulator index:  
  `fullIndex = (xdtaWord << 16) | pdtaWord`

### `pmod`
This chunk exists solely to preserve parser compatibility and contains only the terminal modulator record.

### `pgen`
Same as `pmod`, this chunk includes only the terminal generator record to allow reuse of the `pdta` parser.

### `inst`
- `CHAR achInstName[20]` — second half of the instrument name (combined with `pdta` name for up to 40 characters).
- `WORD wInstBagNdx` — upper 16 bits of the instrument bag index:  
  `fullIndex = (xdtaWord << 16) | pdtaWord`

### `ibag`
- `WORD wGenNdx` — upper 16 bits of the generator index.
- `WORD wModNdx` — upper 16 bits of the modulator index.

### `imod`
The same as `pmod`, includes only the terminal modulator record.

### `igen`
The same as `pgen`, includes only the terminal generator record.

### `shdr`
- `CHAR achSampleName[20]` — second half of the sample name (combined with `pdta` name for up to 40 characters).
- All other fields except `wSampleLink` are unused and set to zero:
  - `dwStart`, `dwEnd`, `dwStartloop`, `dwEndloop`, `dwSampleRate` — these are already 32-bit integers
  - `byOriginalPitch` — unused, set to zero.
  - `chPitchCorrection` — unused, set to zero.
- `WORD wSampleLink` — upper 16 bits of the sample link index:  
  `fullIndex = (xdtaWord << 16) | pdtaWord`
- `SFSampleLink sfSampleType` — unused, set to zero.

---

## Reference Implementation

[spessasynth_core](https://github.com/spessasus/spessasynth_core) and programs based on it (like SpessaSynth or SpessaFont) have support for both reading and writing xdta.

---

## Rationale

- Index limits are increased from 65,536 to **4,294,967,296** by combining the 16-bit `xdta` and `pdta` values into a 32-bit value. 
This ensures that the limit won't ever be hit (as this is also the maximum size of a RIFF chunk).
- Instrument, preset, and sample names can now use up to **40 characters**, improving readability. 
For example: `*Detuned EP 2` can be turned into `*Detuned Electric Piano 2`.
- The format remains **fully backward-compatible** with existing SF2 parsers. 
Legacy software will still be able to parse and use the standard portion of the file.
- `pdta-list` parsers can be reused with minimal changes, reducing implementation complexity for developers.

---

## Appendix: Compatibility Note

If `xdta` is present, but it does not match the `pdta` chunk (like mismatched preset, instrument, sample or zone count), 
it should be ignored and only the `pdta` chunk should be used.


