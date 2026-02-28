# What SFe 4.0 should have been

Below is my proposal for what SFe 4.0 should have been.
This is a release draft, for which I could modify spessasynth and call it a first proper release with reference implementation.

There are also some optional things, but they can be omitted as they are not 100% necessary.

*Note: This is mostly directed to the SFe team. It needs reformatting (more formal tone), but the spec itself is release ready.*

## Design goals

- Only describe changes from SF2.04
- Easy to implement (most important!)
- The point above requires making breaking changes, but it's "close enough" that most code can be reused.
E.g. changing the width from 16 to 32 requires changing one number in the code, but it is technically a breaking change.
- Similar enough to SF2 (i.e. no changing of the synthesis model, core parameter structure, etc.)
- Every change must have a reason

## 64-bit chunks

*Reasoning: Current SF2 files are limited to 4GB which some of them already hit
(for example ApolloGMGS or HiDef (stgiga's SC-88Pro soundfont)). 
I can easily implement this by modifying the RIFF reading and writing function with a single boolean.*

The file may be either use 32-bit or 64-bit chunks. This is indicated by the FourCC of the main RIFF chunk:

- `RIFF` - 32-bit. Standard RIFF header.
- `RI64` - 64-bit. I picked `RI64` instead of `RF64` to avoid confusion with that format, which is a completely different thing.

The only difference between the two is the `ckSize` field in the header.
It's 32-bit LE for `RIFF` (like the regular RIFF spec) and 64-bits for `RI64`.
Note that **this applies to all chunks within the file!**

So the 64-bit structure looks like this:

```c
typedef struct {
    char ckID[4];        // A chunk ID identifies the type of data within the chunk.
    QWORD ckSize;        // The size of the chunk data in bytes, excluding any pad byte.
    BYTE ckDATA[ckSize]; // The actual data plus a pad byte if req’d to word align.
};
```
(the padding byte constraint and all other RIFF rules still apply)

There's no requirement for writing either 32-bit or 64-bit files. The software may only decide to write one version, 
or pick depending on the need of 64-bit sizes.

## FourCC

*Reasoning: Avoid conflict with SF2 files. One if statement modification needed.*

The fourCC instead of `sfbk` is `sfe4`. Descriptive and easy, better than the currently proposed `sfen`

## INFO Chunk

### Text chunks

*Reasoning: Currently INAM and similar chunks have arbitrary limits and are technically limited to ASCII.
This limits localization and more verbose descriptions for sound banks. 
Most players already ignore these chunks and I can implement this easily by replacing the ASCII reader with a UTF-8 one.
Also, I mean, c'mon, it's 2026.*

All text chunks no longer have any limit (other than the RIFF size limit) and have to use the utf-8 encoding.

### ISO-8601 for ICRD

*Reasoning: Currently, the creation date chunk can be anything. 
Polyphone puts localized data there, 
which means that the computer can't read it properly. 
For example, what does `2 stycznia 2025r.` mean? 
The computer has no idea and if the user wants to sort out soundfonts by creation date,
they can't. I can easily implement this*

The ICRD chunk holds an ISO-8601 date or date/time string in UTC. 
Either `YYYY-MM-DD` or `YYYY-MM-DDTHH:MM:SSZ`. No other versions are allowed.
For example `2025-08-17` or `2025-08-17T19:15:25Z`.

Conversion from SF2 may attempt translating localized date, 
and may use current date on failure, 
but that's up to the implementation. The output must be ISO 8601.


### Default modulator chunk

*Reasoning: Currently, the SoundFont standard does not allow us to add default modulators.
The only way to do so is
by adding them to every instrument, which can be a very tedious task
and heavily discourages users from adding extended CC support to their SoundFonts.*

*This is also already a part of SFe 4 and is supported in synths such as fluidsynth or spessasynth!*

A `DMOD` sub chunk within the INFO list.
This chunk is optional, have no length limit and the formatting is exactly the same as `PMOD` or `IMOD` modulator list, that is:

```c
struct sfModList
{
  SFModulator sfModSrcOper;
  SFGenerator sfModDestOper;
  SHORT modAmount;
  SFModulator sfModAmtSrcOper;
  SFTransform sfModTransOper;
};
```

Always multiple of ten bytes and the terminal zero modulator at the end.

#### Default modulator behavior

The behavior is simple: The DMOD chunk replaces all the default modulators.

The default modulator list is replaced at load time, and then it acts exactly like the default SF2 modulator list.
If the DMOD chunk is present but empty (i.e., only the terminal record), the sound bank has no default modulators.

If there is no DMOD chunk, the default modulator list shall be applied.

#### Example behavior

For example, assume a DMOD chunk of two modulators:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

The default modulators for this sound bank will be:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

### Subject chunk

*Reasoning: DLS Parity. DLS files often use this chunk for description.*

A "subject" chunk with a fourCC of `ISBJ`. A text utf-8 chunk, like `INAM` or `ICMT`.

## Sample data chunk

*Reasoning: Currently, all samples are limited to 16 or 24 bit LE with no compression.*

The `sdta` LIST now works differently:
- The `sm24` chunk is removed. (no need as samples can now be 24-bit and more!)
- Samples are still stored in `smpl`, but they no longer are fixed 16-bit LE samples.

### Sample types

Each sample binary data blob is in one of the four formats:

- WAVE - 8-bit, 16-bit or 24-bit Little-Endian PCM, or 32-bit IEE float. *Reasoning: SF2 sample data can be carried over, easiest to read. Also fixes the problem of bit depth.*
- Ogg Vorbis - *Reasoning: SF3 files use this codec (so synths already have the vorbis codec), open with many free implementations.*
- FLAC - *Reasoning Lossless compression, open with many implementations available.*

All of these formats must store mono audio. Their headers are distinctive enough for to detect:

- WAV -> `RIFF:WAVE`
- FLAC -> `fLaC`
- Ogg Vorbis -> `OggS`...`vorbis`

Only WAV is 100% mandatory, all synths must support all encodings of it (similarly to how many of them support SF2 but not SF3).

Synths may reject compressed samples, which ensures that no additional dependencies have to be added if they already chose not to support SF3.

Note: There's no Opus here despite the team considering it as while it's better than Vorbis, 
it feels a bit redundant as FLAC exists for ultimate quality...

Note 2: This doesn't use the revised containerization (i.e. single sample per `smpl` chunk) as it would probably require a major sample code rewrite.

## Preset data chunk

### Preset Header Changes

#### achPresetName

*Reasoning: consistent utf-8 support.*

Encoding now allows utf-8 and the length is doubled to 40 *bytes* to accommodate this change.

#### wBank

*Reasoning: DLS parity, XG support and more! The most important feature of SFe.*

`wBank` field is now holding 3 values, rather than one number representing bank MSB.

The low 7 bits are the bank MSB number.
Bit 7 is the GS/GM drum flag.
The upper 7 bits are the bank LSB number.
So essentially what the current spec proposes.

The proposed way of handling these values is by using the [MIDI Patch system.](https://spessasus.github.io/spessasynth_core/spessa-synth-processor/midi-patch/)

(optional) a neat idea would be to use bit 15 as a toggle for GS/XG sound bank so the players could toggle between them.
Though that's optional.

#### wPresetBagNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.


### Preset Zone Changes

### wGenNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.

### wModNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.

### Preset Modulator Changes

*Reasoning: More potential modulator sources (in the future).*

The `SFModulator` data type (source and secondary source) are now 32-bit rather than 16-bit.
For now the upper 16 bits are reserver and must be ignored.

Note: possibly support NRPN with this?

### Preset Generator Changes

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files (index generators).*

`genAmount` is now 32-bits rather than 16-bits. Note that the sign bit must be correctly converted when converting from SF2.


### Instrument Header Changes

### achInstName

*Reasoning: consistent utf-8 support.*

Encoding now allows utf-8 and the length is doubled to 40 *bytes* to accommodate this change.

### wInstBagNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.

### Instrument Zone Changes

### wInstGenNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.

### wInstModNdx

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files.*

The value is now 32-bit LE rather than 16-bit LE.

### Preset Modulator Changes

*Reasoning: More potential modulator sources (in the future).*

The `SFModulator` data type (source and secondary source) are now 32-bit rather than 16-bit.
For now the upper 16 bits are reserver and must be ignored.

Note: possibly support NRPN with this?

### Instrument Generator Changes

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files (index generators).*

`genAmount` is now 32-bits rather than 16-bits. Note that the sign bit must be correctly converted when converting from SF2.

### Sample Header Changes

*Reasoning: Removing the 65k data limits that can be easily hit with larger SF2 files, consistent UTF-8 support, support for async sample loading.
Most of this structure is already implemented in SF3, so synths like fluidsynth already implemented it.*

Revised structure:

```c
struct sfSample
{
  CHAR achSampleName[40];
  QWORD qwStart;
  QWORD qwEnd;
  DWORD dwLength;
  DWORD dwStartloop;         // Unchanged type
  DWORD dwEndloop;           // Unchanged type
  DWORD dwSampleRate;        // Unchanged type
  BYTE byOriginalPitch;      // Unchanged type
  CHAR chPitchCorrection;    // Unchanged type
  DWORD dwSampleLink;
  SFSampleLink sfSampleType; // Unchanged type
};
```

- `achSampleName` now supports UTF-8, similarly to presets and instruments and has a length of 40 *bytes* maximum.
- `dwStart` is now `qwStart`, 64 bits instead of 32. This is needed since most of the file is the sample data chunk and keeping it as `DWORD` does not increase the 4GB limit.
- `dwEnd` is now `qwEnd`, 64 bits instead of 32. For the same reason as above.
- `dwLength` is a new field, the length of the uncompressed sample data (in samples).
This allows players to allocate the space needed for the samples and "play" them 
while they are being uncompressed in the background (this behavior is not required, but reusing this field makes it possible).
- `dwStartloop`, similarly to SF3, now is relative to the start of the uncompressed sample data (in samples).
- `dwEndloop`, similarly to SF3, now is relative to the start of the uncompressed sample data (in samples).
- `wSampleLink` is now `dwSampleLink`, 32-bit index instead of 16.

(optional) sfSampleType could have linked and rom samples forbidden from writing and reserved for future purposes.

## New Generators

*Reasoning: The generators use values above endOper to avoid conflicts with unofficial extensions.*

#### vibLfoToVolume

*Reasoning: GS/XG parity (they use 2 LFOs and both of them can do everything. For example see SC-8850 manual p.239)*

Essentially modLfoToVolume but for the vibrato LFO

- Number: `61`
- Unit: cent fs
- Min: -12000 = 10 oct
- Max: 12000 = 10 oct
- Default: 0 = None

This is the degree, in centibels, to which a full scale excursion of the Vibrato
LFO will influence volume. A positive number indicates a positive LFO excursion
increases volume; a negative number indicates a positive excursion decreases
volume. Volume is always modified logarithmically, that is the deviation is in
decibels rather than in linear amplitude. For example, a value of 100 indicates that
the volume will first rise ten dB, then fall ten dB.

#### vibLfoToFilterFc

*Reasoning: GS/XG parity (they use 2 LFOs and both of them can do everything. For example see SC-8850 manual p.239)*

Essentially modLfoToFilterFc but for the vibrato LFO

- Number: `62`
- Unit: cent fs
- Min: -12000 = 10 oct
- Max: 12000 = 10 oct
- Default: 0 = None

This is the degree, in cents, to which a full scale excursion of the Vibrato LFO
will influence filter cutoff frequency. A positive number indicates a positive LFO
excursion increases cutoff frequency; a negative number indicates a positive
excursion decreases cutoff frequency. Filter cutoff frequency is always modified
logarithmically, that is the deviation is in cents, semitones, and octaves rather than in
Hz. For example, a value of 1200 indicates that the cutoff frequency will first rise 1
octave, then fall one octave.

#### variationEffectsSend

(optional)

This could be a generator like `reverbEffectsSend` or `chorusEffectsSend`, 
except it controls the XG variation or GS delay (CC#94) send level.

#### shutdownModEnv

*Reasoning: DLS Parity.*

- Number: `63`
- Unit: timecent
- Min: -12000 = 1 msec
- Max: 8000 = 100sec
- Default: -2000 ([see this issue](https://github.com/FluidSynth/fluidsynth/issues/1467))

The shutdownModEnv generator determines the duration of the release segment when a key-exclusive event occurs.
It behaves identically to the releaseModEnv generator, but is only invoked when an
exclusive event causes the shutdown of a voice.

#### shutdownVolEnv

*Reasoning: DLS Parity.*

- Number: `64`
- Unit: timecent
- Min: -12000 = 1 msec
- Max: 8000 = 100sec
- Default: -2000 ([see this issue](https://github.com/FluidSynth/fluidsynth/issues/1467))

The shutdownVolEnv generator determines the duration of the release segment when a key-exclusive event occurs.
It behaves identically to the releaseVolEnv generator, but is only invoked when an
exclusive event causes the shutdown of a voice.


## Modulator Changes

SF2 conversions *must* write the SF2 default modulators 
into the DMOD chunk. This maintains the existing behavior fully.

### Note about SFe extra modulators

They are currently in the SFe spec. That is:

- CC#92 (tremolo) to LFO to volume
- Poly to vibrato (removed in v4.2)

I removed them as it is not accurate to GS/XG at all. So they are not included here.

### Velocity to cutoff

*Reasoning: Most synths already omit it and it is not consistent.
Here's the reason copy-pasted from fluidsynth:
The default "note velocity to filter cut off" modulator is 
inconsistently defined by the spec and is therefore actively disabled by fluidsynth. 
Generally, this is no problem, as many people feel that musically it doesn't make sense anyway. 
Therefore, it is usually not missed by users.*

Properly removed.

### Channel pressure to vibrato

*Reasoning: Neither GS nor XG do that, they often use CC to param sysExes and then use it.
[Example](https://files.catbox.moe/sc50nn.mid)*

Removed.

### Effect send levels

*Reasoning: XG/GS/BASSMIDI/spessasynth parity.*

Amount increased from 200 to 1000.

### Others
*Reasoning: XG/GS parity, used very often!*

[This table.](https://spessasus.github.io/spessasynth_core/sound-bank/modulator/#custom-modulators)

Notes:
- Filter cutoff: I've changed the amount to 9600 as it's very accurate to SYXG50 (at least for the "door squeak" sample).
- Filter resonance: I've changed the amount to 200 for the same reason. 
This modulator should not affect the DSP gain like the SF2 specification requires. 
This is done because neither XG nor GS reacted that way to CC 71. 
Other than that, the filter is fully to the spec. 
All other modulators/generators affect the gain as the spec requires.
- Attack time: this needs more work as attack is often set to -12000 (default), so this modulator doesn't make any difference. But we can keep it for now.

## Conclusion

I believe that replacing the current specification (including the SFty and other chunks I don't understand)
with this simplified version will help other devs (esp. fluidsynth) adopt this format.

I could modify spessasynth relatively easily for it to support this,
so SFe would finally have the reference implementation it needs and finally properly release.