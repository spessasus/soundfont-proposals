# Default Modulator Proposal
This proposal adds support for default modulators in the SoundFont2 format.

Originally created by me in https://github.com/davy7125/polyphone/issues/205.

## The Problem
Currently, soundfont does not allow us to add default modulators.
The only way to do so is by adding them to _every instrument_ which is absolutely terrible.
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

Always multiple of ten bytes and the terminal modulator at the end.

This chunk would preferably appear as the last chunk within the INFO list.

### Default modulator behavior
The behavior is simple: The DMOD chunk replaces all the default modulators.

The default modulator list is altered at load time, and then it acts exactly like the default SF2 modulator list.

### Example behavior

For example, assume a DMOD chunk of two modulators:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

The default modulators for this soundfont will be:

- MIDI CC 1 to vibratoToPitch, linear unipolar positive, no controller, amount 100.
- Poly Pressure to vibratoToPitch, linear unipolar positive, no controller, amount 50.

That's it!

## A test SoundFont
There's a [test soundfont](DMOD%20Test%20SoundFont_v2.sf2) available for testing. It has just one default modulator:
- Invert the velocity to initialAttenuation generator

This disables all the other modulators, such as pitch wheel, pan or volume.

Currently, this chunk is supported and read by SpessaSynth. Its implementation complies with this proposal.

## Rationale
Sfspec24 section 3.2:
> The SoundFont 2 specification requires that implementations ignore unknown sub-chunks within the INFO-list chunk. 

This allows us to add a new chunk without violating the spec.

This seems to be the perfect solution because:
1. Doesn't violate the SF2 spec (foreign INFO chunks shall be ignored)
2. Default modulators, yay!
3. Code from the IMOD and PMOD parsers can be reused.
4. It should be easy to implement for most players.
