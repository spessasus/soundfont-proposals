# Default Modulator Proposal
This proposal adds support for default modulators in the SoundFont2 format.

Originally created by me in https://github.com/davy7125/polyphone/issues/205.

## The Problem
Currently, soundfont does not allow us to add default modulators. The only way to do so, is by adding them to _every instrument_ which is absolutely terrible.
## The Solution
I propose to add a `DMOD` sub chunk within the INFO chunk. The formatting would be exactly the same as `PMOD` or `IMOD` modulator list, that is:

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

### Default modulator behavior
> Note: this assumes the reader is familiar with what sf2 specification defines as an "identical" modulator.

- If a modulator is identical to the defined SF2 default modulator, it overrides the modulator in the default modulator list
- If a modulator is identical to the defined SF2 default modulator and has the `transform` field set to 0, it removes it from the default modulator list.

The default modulator list has to be estabilished at load time

## Rationale
Sfspec24 section 3.2:
> The SoundFont 2 specification requires that implementations ignore unknown sub-chunks within the INFO-list chunk. 

This allows us to add a new chunk without violating the spec.

This solution does not violate the file format standard and is easy to implement.
