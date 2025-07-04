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
    * [The extension does not violate the SoundFont Specification](#the-extension-does-not-violate-the-soundfont-specification)
    * [Default Modulators are Cool](#default-modulators-are-cool)
    * [Easy To Implement](#easy-to-implement)
    * [Unambiguous](#unambiguous)
    * [Simple behavior](#simple-behavior)
    * [Encouraging more modulators](#encouraging-more-modulators)
    * [Better CC Extensions](#better-cc-extensions)
<!-- TOC -->

## The Problem
Currently, the SoundFont standard does not allow us to add default modulators.
The only way to do so is
by adding them to _every instrument,_ which can be a very tedious task
and heavily discourages users from adding extended CC support to their SoundFonts.

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
If the DMOD chunk is present but empty (i.e., only the terminal record), the SoundFont has no default modulators.

If there is no DMOD chunk, the default SoundFont modulators shall be applied.

### Example behavior

For example, assume a DMOD chunk of two modulators:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

The default modulators for this SoundFont will be:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

That's it!

### A test SoundFont
There's a [test SoundFont](DMOD%20Test%20SoundFont_v2.sf2) available for testing. It has just one default modulator:
- Invert the velocity to initialAttenuation generator

This disables all the other modulators, such as pitch wheel, pan or volume.

Currently, this chunk is supported and read by SpessaSynth. Its implementation complies with this proposal.

### Editing Default Modulators
My [SpessaFont](https://spessasus.github.io/SpessaFont) SoundFont editor allows reading,
editing and writing default modulators.

## Rationale

### The extension does not violate the SoundFont Specification

> 10.2 Unknown Chunks
> 
> In parsing the RIFF structure, unknown but well-formed chunks or sub-chunks may be encountered. Unknown chunks
within the INFO-list chunk should simply be ignored. Other unknown chunks or sub-chunks are illegal and should be
treated as structural errors.

Unknown INFO chunks are allowed and will be ignored by incompatible SF2 synthesizers.

### Default Modulators are Cool
Currently, SoundFont's design is discouraging SoundFont creators from using modulators
 for extending CC support (like brightness or filter resonance),
  since having to add them to _all the instruments_ can be very time-consuming.

### Easy To Implement
Since the structure of DMOD is exactly the same as PMOD or IMOD, 
the parsers can be reused.

### Unambiguous
When the SoundFont has its own modulators specified, it's clear for them. 
No more of the confusing velToFc modulator! 
If the creator wants the modulator, 
it is directly declared in the SoundFont and all the confusion is avoided.

### Simple behavior
No need for that confusing addition with the preset and instrument modulators. 
Just a drop-in replacement with the complete default modulator definition!

### Encouraging more modulators
SoundFont editors
 could automatically add an extended list of modulators for a broader CC support 
 [(my example)](https://github.com/spessasus/spessasynth_core/wiki/Modulator-Class#default-modulators). 
 If the creator doesn't want them or wants to change them, no problem! 
 Just edit them, no need to mess with every instrument.

### Better CC Extensions
DMOD removes the ambiguity of modulator extensions: 

For example, 
`SpessaSynth` and `BASSMIDI` both support brightness CC by default, 
yet their curves and amount are a bit different, 
resulting in the same MIDI file sounding different on both synthesizers. 

If all synths implement DMOD, 
they will all use DMOD the same way and all the extensions 
coded in will work exactly the same. 
For example, 
Yamaha XG inspired SoundFonts can have the XG Brightness CC curve baked in globally, 
making it easy to adjust if necessary.
Created by spessasus