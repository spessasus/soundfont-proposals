# NRPN Modulator Sources Proposal

This proposal allows NPRNs to be used with SF2 modulators.

## The Problem
Currently, SF2 modulators only allow modulating Continuous Controllers.
This can be limiting for soundfont creators looking to create, for example, G
S-compatible soundfonts. For example, many MIDI files use NRPN messages for effects such as vibrato.

Another problem might be adding support for RPNs implemented after the SF2 standard was created, 
such as modulation depth range.

## The Solution

I propose to and `PNMM` (Parameter-Number-Modulator-Map) chunk,
which maps the given parameter numbers to a Continuous Controller.

### Chunk Structure
The chunk would always have a length multiple of 4.

Each four bytes represent a single parameter-to-cc connection. 
There can be an unlimited number of connections.

Connection structure is as follows:
- `flags` - `char` - one byte, two flags:
  - bit 0 (least significant): 
    - if set to `0`, 
    the mapping responds to a Data Entry CC change only if the state is set to Non-Registered Parameter. 
    - If set to `1`, it only responds to Registered Parameter.
  - bit 1:
    - `0` indicates that only Data Entry MSB is mapped.
    - `1` indicates that only Data Entry LSB is mapped.
- `parameterNumberMSB` - `char` - the parameter number's MSB to connect to a CC. Note that both values must match. A value of 127 is not allowed.
- `parameterNumberLSB` - `char` - the parameter number's LSB to connect to a CC. Note that both values must match. A value of 127 is not allowed.
- `controllerNumber` - `char` - the CC number to map a given parameter to. Ranges from 0 to 127.

## Example chunk
- `DMOD` FourCC
  - size: `8` meaning two connections defined.
    - `flags`: `0` - uses NRPN, only uses Data Entry MSB.
    - `parameterNumberMSB`: `1` - NRPN MSB must be equal to 1.
    - `parameterNumberLSB`: `8` - NRPN LSB must be equal to 8.
    - `controllerNumber`: `76` - the CC#76 will be altered.
    - `flags`: `1` - uses RPN, only uses Data Entry MSB.
    - `parameterNumberMSB`: `1` - RPN MSB must be equal to 0.
    - `parameterNumberLSB`: `8` - RPN LSB must be equal to 5.
    - `controllerNumber`: `55` - the CC#5 will be altered.

This shows two connections:
- `NRPN 1 8` (A.K.A vibrato rate) to `CC#76` via `Data Entry MSB`.
- `RPN 0 5` (A.K.A. modulation depth range) to `CC#56` via `Data Entry MSB`.

### Example behavior
For these two connections, the behavior is as follows:
- received an NRPN MSB: 1. Software remembers the value.
- received an NRPN LSB: 8. Software remembers the value.
- received Data Entry MSB: 40. Software internally calls Controller Change 76 to 40.
- received Data Entry LSB: Ignored.

- received an RPN MSB: 0. Software remembers the value.
- received an RPN LSB: 5. Software remembers the value.
- received Data Entry MSB: 3. Software internally calls Controller Change 55 to 3.
- received Data Entry LSB: Ignored.

### Note
A connection using a defined RPN (like coarse tuning) or NRPN (for the software that supports it)
disables the default behavior.

## Rationale
Sfspec24 section 3.2:
> The SoundFont 2 specification requires that implementations ignore unknown sub-chunks within the INFO-list chunk.

This allows us to add a new chunk without violating the spec.

Other than that:
- soundfont creators can add support for various systems such as before mentioned GS or even XG.
- Allows to implement the missing modulation depth range RPN.
- Allows modifying the RPN behavior if needed.
- If the implementation already has its own support for these,
it will override that implementation with the soundfont-defined one.
- It Does not violate the soundfont specification or file structure.
