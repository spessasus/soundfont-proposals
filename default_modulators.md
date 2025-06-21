# Default Modulator Proposal
This proposal adds support for default modulators in the SoundFont2 format.

Originally created by me in https://github.com/davy7125/polyphone/issues/205. Updated on 08-06-2025.

## Table Of Contents
<!-- TOC -->
* [Default Modulator Proposal](#default-modulator-proposal)
  * [Table Of Contents](#table-of-contents)
  * [The Problem](#the-problem)
  * [The Solution](#the-solution)
    * [Default modulator behavior](#default-modulator-behavior)
    * [Example behavior](#example-behavior)
    * [A test SoundFont](#a-test-soundfont)
    * [Editing Default Modulators](#editing-default-modulators)
  * [Rationale](#rationale)
<!-- TOC -->

## The Problem
Currently, soundfont does not allow us to add default modulators.
The only way to do so is
by adding them to _every instrument,_ which can be a very tedious task
and heavily discourages users from adding extended CC support to their banks.
## The Solution
I propose to add a `DMOD` sub chunk within the INFO list. 

This chunk would be optional, have no length limit and the formatting would be exactly the same as `PMOD` or `IMOD` modulator list, that is:

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

### Default modulator behavior
The behavior is simple: The DMOD chunk replaces all the default modulators.

The default modulator list is replaced at load time, and then it acts exactly like the default SF2 modulator list.
If the DMOD chunk is present but empty (i.e., only the terminal record), the bank has no default modulators.

If there is no DMOD chunk, the default soundfont modulators shall be applied.

### Example behavior

For example, assume a DMOD chunk of two modulators:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

The default modulators for this soundfont will be:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

That's it!

### A test SoundFont
There's a [test soundfont](DMOD%20Test%20SoundFont_v2.sf2) available for testing. It has just one default modulator:
- Invert the velocity to initialAttenuation generator

This disables all the other modulators, such as pitch wheel, pan or volume.

Currently, this chunk is supported and read by SpessaSynth. Its implementation complies with this proposal.

### Editing Default Modulators
My yet to be finished [SpessaFont](https://github.com/spessasus/spessafont) sound bank editor allows reading,
editing and writing default modulators.

Note that the editor is still unfinished, but it does provide full editing support for the DMOD chunk for now.
There is no on-screen keyboard, so testing with the SpessaSynth web app is recommended.

## Rationale
Sfspec24 section 10.2:

> 10.2 Unknown Chunks
>
> In parsing the RIFF structure, unknown but well-formed chunks or sub-chunks may be encountered.
> Unknown chunks within the INFO-list chunk should simply be ignored.
> Other unknown chunks or sub-chunks are illegal and should be treated as structural errors.

This allows us to add a new chunk without violating the spec.

This seems to be the perfect solution because:
1. It does not violate the SF2 spec. (foreign INFO chunks shall be ignored)
2. Default modulators, yay!
3. It will make modulators way easier to use, making users want to use the more often.
4. Code from the IMOD and PMOD parsers can be reused.
5. Inconsistently defined 2.01 and 2.04 modulators are resolved if the chunk is present.
6. Simple implementation: replace the default modulator list if encountered.

Created by spessasus