# Instruction Format

Ych16 instructions are variable-length. They do not adhere to specific "size forms", rather, the instruction is gradually reasoned about, until there is nothing left to discuss and/or reason about, by which point you will have reached the end of the instruction. Logical rules and instruction stream work together in this process of reasoning. Ych16 instructions range from **1 byte** to **13 bytes** at most (this number can change with ISA revisions over time, but this is what `ISA Rev. A` instructions peak at). This document will explain in-depth how Ych16 instructions look like.

The general Ych16 instruction looks like the following:  
`[Prefix Byte (optional)] [Opcode Byte (mandatory)] [Operand Bytes (depends on opcode, optional/mandatory)]`

## The Prefix Byte; What is it & How is its presence known?

Ych16 instructions use the newer **Unified Prefix Byte (`UPB`)** system over the legacy prefix stacking system. This makes it way simpler to reason about an instruction, because there can ever only be 1 prefix byte per instruction.

If the first fetched byte at the beginning of an instruction decode is `0?h aka range [00-0F]`, i.e. the **high nibble** of the byte is `0000b`, that fetched byte is the Prefix Byte, and the very next byte that follows the prefix byte is always the opcode byte. If the high nibble is not zero, the fetched byte is the opcode.

Example Instruction Streams:
1. `08 30 11 ...` -> This instruction has the UPB as the very first byte is within range `[00-0F]`, with the high nibble `0000b`.
2. `30 11 ...` -> This instruction does not have the UPB. The very first byte is the opcode.

As stated previously, you cannot have multiple prefix bytes per instruction, as the decoder will treat the next byte after the prefix as the opcode, not as another prefix. If you put another byte in range of `[00-0F]` after the prefix, the decoder will raise `Nonsensical Instruction Exception (NIE)` as no valid Ych16 instruction has such opcodes.

The Prefix Byte has the following layout:

| Bit Offset | Tag |      Prefix Name      | Description |
| :--------: | :-: | :-------------------: | :---------: |
|    `7-4`   |  -  |           -           | High nibble of `0000b` denoting the UPB. |
|     `3`    |  R  |         Repeat        | Repeats the instruction `RC` times with well-defined architectural semantics. |
|     `2`    |  F  | Preserve Status Flags | Ensures that the **Processor Status Word (`PSW`)**'s *ALU flags* (bits 3-0) are kept intact. This means that the PSW bits 3-0 after the instruction execution will exactly match the ones before execution & retirement. |
|     `1`    |  T  | Transaction Size Alter | Inverts the transaction size implied by the opcode. This means that an opcode with 8-bit TSZ will flip to 16-bit, and an opcode with 16-bit will flip to 8-bit. |
|     `0`    | DOP | Destination-Operand Port Override | Alters destination-operand to target port address space instead of register/memory. |

### Repeat Prefix Semantics

Repeats an instruction RC times. For each repetition, the instruction is retired and only then the repetition happens, with the operands reevaluated on each execution. The order is:  
1. Test RC. If zero, repetition done, bail out. If nonzero, proceed to step #2
2. Execute
3. Retire
4. Decrement RC by 1
5. Go Back to Step #1

The repeat itself is not wholly atomic, whereas individual repetitions retain the instruction's atomicity. This means repetition #1 may succeed and that execution is retired, whereas repetition #2 results in an exception so that specific repetition is not retired and the process stops at execute and doesn't progress to retire. This also means that upon return from exception handler if at all, if the exception handler had potentially fixed the issue, the repeat semantics continue flawlessly in case that the architectural state was preserved.  
Decrementing/Testing RC is internal and does not reflect anything in architectural state, except for RC's actual value changing.  
Self-modifying code behavior is undefined. Do not overwrite the repeating instruction's bytes.

### Operand Port Override Prefix Family

Normally, you'd think an UBP (Unified Prefix Byte) of value `00` indicates a no-prefix prefix byte. Well, as you can see from that sentence, that absolutely makes no sense. So the Ych ISA *repurposes* that encoding, alongside the UPB prefix `xxx1` (bit 0 of UPB low nibble set), to alter what the operand machinery does for a specific purpose.

The **Operand Port Override** system allows for the repurposing one of the operands to address the **port address space** rather than the traditional register/memory locations. This is important because it unlocks orthogonal operations to be conducted from and to peripherals and such without having to treat port I/O as an afterthought, so it integrates directly into the already existing powerful instruction set, operand and addressing machinery the Ych architecture has to offer.

Only one operand may ever be port-overridden in an instruction. When a port-overridden operand is present, the instruction can no longer use the three-operand extension mechanism offered by OTO DB and OTO EB. Only solo-operand and dual-operand forms are valid, as the standard three-operand escape sequence cannot trigger due to the second condition, observable by checking out [this section for context-relevant encoding #2](#context-relevant-uod-encodings).

When an operand is port-overridden, its raw value (for registers) or its effective-address (EA, for memory) will be the port to target within the port address space, and the interaction still respects the Transaction Size semantics of the instruction. This means the TSZ of the instruction is also the size of the value to read/store from/to the port.

This prefix is encoded in two manners:
| UPB Low Nibble Encoding | Description of the Specific Instance |
| :---------------------: | :----------------------------------: |
|       `0000` (SOP)      | The source operand of the instruction will be used to target the port address space, and the source will therefore come from the port. |
|       `xxx1` (DOP)      | The destination operand of the instruction will be used to target the port address space, and the destination will therefore be the port, so if a peripheral is present at that address, it will receive the store. |

There is a bit of an asymmetry when it comes to the Source/Destination Port Override prefixes. When you encode a SOP prefix, you cannot encode any other prefixes with the instruction, so no `RPT`. Whereas when you encode DOP, you can still encode other prefixes like `RPT`. This is like this due to two reasons: source port override already works by hijacking a very specific pattern (`0000`) which would be a no-op prefix, so it literally cannot encode anything without mutating into an ordinary UPB. Whereas DOP works by encoding bit 0 of the UPB as set, so the other prefixes are still available to encode. The other reason is that the destination was chosen for this role specifically, particularly for the case of repeat-storing to a frame buffer within the port address space by utilizing `RPT` and `RC` alongside `CRA` operand machinery.

Crucially, for a port-overridden operand, the operand semantics for memory operands shift slightly, as in, their effective-address will directly be the port address, with no memory read operation incurring, it won't do "read memory at EA, use value at that address as the EA for the port", no, the EA result is directly the port address. Yes, this also means register-direct and register plain-indirect operands end up being the same thing. However, register-direct makes it possible to name all registers, so it has that advantage, whereas Memory operands are more useful when you need to do actual EA computation for the port address.

Using Immediate instead of Register/Memory for a port-overridden operand is also normal. The immediate value will be the EA of the port address. So an Imm16 and AMA operand, when port-overridden, also mean the same thing, just like how register-direct and register plain-indirect do so.

When an operand is port-overridden, the CommittableTo requirement for that operand vanishes. That is because the Operand Class becomes irrelevant as the CPU will eventually knock the door on the port address space (not the immediate for example), and the port address space satisfies CommittableTo.

Before we move on, here is a table for what each Operand Class does for the eventual port EA calculation, a quick cheatsheet, if you will.
| Operand Class (OC) | Description |
| :----------------: | :---------: |
|     Register       | The literal raw value stored within the register is the literal EA of the port to address. |
|      Memory        | The literal EA of the memory address is the literal EA of the port to address. |
|     Immediate      | The literal immediate value is the literal EA of the port to address. |

## The Opcode Byte

The opcode is essential in terms of defining how the next bytes of the instruction stream following it to be interpreted. It specifies what sort of operand configuration is expected, if any. The opcode establishes the base requirements, whereas those requirements can then enforce other requirements in a cascading manner, though it will always exhaust logically. The opcode also denotes the **Transaction Size (`TSZ`)** of the instruction inherently.

## Operands & Operand Classes

The Ych16 ISA has a total of 3 Operand Classes (OCs) that any sort of operand will eventually resolve into:
- Register
- Memory
- Immediate

There is one crucial concept to know, and it's `CommittableTo`. This is a named requirement for Ych operands. For an opcode to satisfy it, it must be something that can store values. Operand Classes of **Register** and **Memory** both satisfy this, whereas **Immediate** does not. Because you can store values in registers and memory, whereas immediates are just literal values present within the instruction stream. This rule must be known about as some instructions also demand the source operand be CommittableTo as they treat the operands not as destination and source but as two orthogonal operands.

As stated previously, the opcode establishes the basic pattern. It will always use one of these operand templates for the operands it expects:

| Operand | Operand Class | Expected Fetch |         Description         |
| :-----: | :-----------: | :------------: | :-------------------------: |
|  n/a    |      N/A      |      N/A       |     No Operand Expected     |
|  shd    |    Memory     |    ShD Byte    |    Shorthand Displacement   |
|  ama    |    Memory     | Immediate Word | Absolute Memory Addressing  |
|  cra    |    Memory     |     CRA DB     | Complex Register Addressing |
|  imm8   |   Immediate   | Immediate Byte | 8-bit immediate, sign-extended to word-size |
|  imm16  |   Immediate   | Immediate Word | 16-bit native-size immediate |
|  oto    |      TBD      |     OTO DB     | Operand-to-Operand orthogonal operand description system |

Opcodes can use any combination of operand templates, up to three of them. The only exception to this rule are `N/A` and `OTO`, they can only be present once and be the only thing present. N/A communicates that this opcode does not expect any operands at all, so that instruction only needed its opcode to be present (except for the optional prefix) and decode will be finished immediately.  
OTO communicates that this opcode uses the `Operand-to-Operand Descriptor Byte (OTO DB)` to describe its operands in an orthogonal way. When OTO is present, it describes all operand grammar of the instruction by itself. For example, using ShD is static and fixed, decoder will always expect ShD. However, the OTO DB encodes what to even expect. OTO DB can describe that for the source operand, it wants an immediate or AMA or CRA and it wants ShD or CRA for destination. Unique to the OTO system, it also is able to describe register-direct and register plain-indirect addressing modes. Any instruction involving registers in direct and plain-indirect form, when they are not inherent to the opcode, mandates the presence and usage of its OTO opcode variant (and perhaps that may already be the only one) because nothing else is able to describe them, plus it also makes the instruction fully operand-orthogonal.

## The Concept of the Transaction Size (TSZ)

The TSZ controls how the processor interacts with its operands and it can also alter the interpretation of some operands, the prominent example being the OTO DB. Regardless, there are two transaction sizes in the Ych16 ISA:

1. Word (16-bits)
2. Byte (8-bits)

Word is usually the default. Here's what the TSZ affects:
1. The granularity in which operand memory transactions (both read and write) are conducted in and the security enforcements
2. The ALU operand size used when conducting ALU operations between the operands and the size of the resulting value and flags
3. The interpretation of the OTO DB UODs

The **Address Size** remains the exact same as the word-size no matter what. TSZ is only relevant to and with the operands.

## The Concept of Finalized Operand State

Ych16 instructions rely on two logical operands; the **source** and the **destination**. That's why you'll see that most instructions work with one or two operands. However, there is a bit of a hack-around with this two logical operand system, and it's about the destination. With OTO addressing, you're able to separate the destination operand to two concepts: the **destination-facade (`DF`)** and the **destination-commit (`DC`)**. This is essentially a hacky way to integrate three-operand addressing system on top of an ISA built as two-operand. The logic is that when the instruction wants to read the destination operand, it will read the destination-facade. When it wants to write/commit to the destination operand, it will write/commit to the destination-commit. This essentially makes it so that the source operand and the destination-facade operand act as two sources (with destination-facade acting as LHS and the source acting as the RHS as usual, since in the normal two-operand system the destination acts as the LHS and the source acts as the RHS), while the destination-commit is the destination. The destination-commit is never used for any sort of computation. These semantics do not disappear when an instruction is only using two-operands, it simply means that both the destination-facade and the destination-commit are the same operand, collapsing into one unified logical destination operand.

Do note that opcodes can explicitly forbid the three-operand extension as an inherent property, like how `loa - Load` and `swa - Swap` does, because having two operands with them is nonsensical, which is why the decoder will raise `Nonsensical Instruction Exception (NIE)`. Always refer to the instructions' documents which will always document the eligibility.

## The Rule of Decode Order: Defining the Operand Decode Order without Ambiguity

The Ych ISA follows a very strict decode order since its cascasing variable-length instruction format mandates rigid rules around it. This is where the **Rule of Decode Order** comes from. The decoder will always decode the operands in this order:
1. Destination-Commit (`DC`)
2. Destination-Facade (`DF`)
3. Source (`SRC`)

If we are talking about a dual-operand situation where both DF and DC are unified into a single unified operand, the order is still the same, but obviously there is just one destination operand to decode:
1. Destination (`DST`)
2. Source (`SRC`)

Now, order has been stated, but the how of it hasn't. The rule is very simple, the decoder has to **fully resolve an operand including all of its fetchmusts** before it progresses to the other operands, if any. What do I mean by this? Suppose an ADD instruction with two AMA operands, one is the destination address and one is the source address. What the decoder will do is that it will first fetch the Immediate Word for the destination operand from the instruction stream following the opcode/OTO DB. Since it's known that the destination is AMA, there is nothing left to fetch for AMA anymore, therefore the destination operand has fully been resolved, and only then will the decoder fetch the Immediate Word for the source operand. Another example is CRA. Suppose DST is CRA, SRC is something else. Decoder will first fetch the CRA for DST as it always starts decoding DST first. It will fetch the CRA DB. Now what happens depends on the CRA DB itself. It might report that there is no displacement, in which case the operand has been fully resolved as there is nothing left to fetch for that operand anymore, so the decoder will move onto source. If that CRA byte were to specify Disp8/16 instead, the decoder would then fetch Immediate Byte/Word from the instruction stream under destination's name, and only then move onto the source operand.

### CRA DB (Complex Register Addressing Descriptor Byte) Layout

Default mode (indexed):

| Bit Offset | Field |        Full Name         | Description | Encode |
| :--------: | :---: | :----------------------: | :---------: | ------ |
|     7-6    |  D/I  | Displacement/Indexedness | Encodes displacement validity, if so, size. Also able to alter into nonindexed form. | • `00` : No Disp<br> • `01` : Nonindexed Mode<br> • `10` : Displacement 8-bits<br> • `11` : Displacement 16-bits |
|     5-4    |  PTR  | Base Register            | The base register. | Base Register ID. |
|     3-2    |  IDX  | Index Register           | The scaled index register. | Index Register ID. |
|     1-0    |  MUL  | Scale Factor             | Index scalar factor. | Sequentially from lowest to highest, `{×1,×2,×4,×8}`. |

Nonindexed mode (D/I = `01`):

| Bit Offset | Field |        Full Name          | Description | Encode |
| :--------: | :---: | :-----------------------: | :---------: | ------ |
|     7-6    |  ---  | Mode Alter                | Value that caused nonindexed mode. | `01` |
|     5-4    |  PTR  | Base Register             | The scaled base register. | Base Register ID. |
|     3-2    |  D/R  | Displacement/Relativeness | Encodes displacement validity, if so, size. Also able to alter into RIP-relative form. | • `00` : No Disp<br> • `01` : RIP-relative Mode<br> • `10` : Displacement 8-bits<br> • `11` : Displacement 16-bits |
|     1-0    |  MUL  | Scale Factor              | Base scalar factor. | Sequentially from lowest to highest, `{×1,×2,×4,×8}`. |

Do note that the main reason Nonindexed Mode exists is that so `Base + Displacement` EA can be achieved, which is doable via this mode and a scale factor of ×1. However nonindexed mode can be very useful in some scenarios as well, so do note fact that it fully exists.

RIP-relative mode (D/I = `01` && D/R = `01`):

| Bit Offset | Field |        Full Name          | Description | Encode |
| :--------: | :---: | :-----------------------: | :---------: | ------ |
|     7-6    |  ---  | Mode Alter                | Value that caused nonindexed mode. | `01` |
|     5-4    |  ---  | Reserved                  | Reserved, must be canonically zero. | `00` |
|     3-2    |  ---  | Mode Alter                | Value that caused RIP-relative mode. | `01` |
|     1-0    |  ---  | Reserved                  | Reserved, must be canonically zero. | `00` |

This always produces byte `44h` due to its nature. It also mandates a forced 16-bit displacement.  
RIP-relative is instruction-start relative, not instruction-end relative. The displacement must be relative to the first byte of the instruction (UPB/opcode).  
**All** CRA displacements are **signed**. 8 bit displacements are **sign-extended** to 16-bits.

##### EA (Effective Address) for Each CRA Mode

|     Mode     | Effective Address (`EA`) Formula |
| :----------: | :-----------------------------: |
|    Indexed   |  Base + (Index × Scale) + Disp  |
|  Nonindexed  |       Base × Scale + Disp       |
| RIP-relative |           RIP + Disp            |

Assume `0` for Disp element of the formula if no displacement exists.

### ShD (Shorthand Displacement)

ShD is a compact addressing primitive that is useful for accessing reasonably-offseted stack locals and struct fields. It can displace up to `+64` bytes from a base register. It is denser than `CRA` + 8-bit displacement because that clocks at 2 bytes of encode while ShD is only one byte of encode. Code density is the reason of its existence.

| Bit Offset | Field Name | Description | Encodes |
| :--------: | :--------: | :---------: | :-----: |
|     7-6    |     PTR    | Base register. | Base register ID. |
|     5-0    |     DIS    | 6-bit **unsigned** displacement. | Actual displacement, minus one. This is because `000000` is treated as `+1` displacement, so counting starts from `1`. This phenomenon is called the **Bias-of-1**, a quick way to remember the rule. |

Any negative displacement or positive displacement exceeding `+64` must go through `CRA`'s nonindexed form with preferred displacement size encoded in `D/R`.  
**DIS** is naturally **zero-extended** as it is an unsigned integer in this instance.

### OTO DB (Operand-to-Operand Descriptor Byte) Layout

The OTO DB is split into two nibbles of nearly the same structure except for a few contextual reinterpretations, and that structure is named a **Unit Operand Descriptor (`UOD`)**. The high nibble of the DB denotes the Destination UOD, the low nibble of the DB denotes the Source UOD.

| Bit Offset |  Field  |
| :--------: | :-----: |
|    `7-4`   | DST UOD |
|    `3-0`   | SRC UOD |

The DST UOD must always satisfy the named requirement **CommittableTo**. When Three-Operand Escape is used, that responsibility is tagged onto Destination-Commit UOD instead.

The base UOD encoding is as follows:

| Bit Pattern |         Operand         | Operand Class | Description |
| :---------: | :---------------------: | :-----------: | :---------: |
|    `xxx1`   |     Register Direct     |    Register   | Register direct operand and addressing mode. The LSB being set denotes this mode. The upper 3 bits `xxx` encode the GPR ID, check [this document](/Registers.md#registers) out for more. Do note that `1111` may or may not be included here, you must refer to [this section](#context-relevant-uod-encodings). |
|    `1111`   | Context Relevant Encoding #1 | TBD | Refer to [this section](#context-relevant-uod-encodings). |
|    `1xx0`   | Register Plain-Indirect |    Memory     | Register plain-indirect operand. The register's value will be the `EA`. The LSB being clear and the MSB being asserted denotes this mode. The middle 2 bits `xx` encode the base register ID, check [this document](/Registers.md#registers) out for more. |
|    `0xx0`   |       Fetchmusts       |       TBD      | Fetchmusts get their name from the fact that the decoder must fetch at least one (potentially many more) byte(s) in order to fully resolve what this operand is, but the context and the encoding is enough to at least classify the operand i.e. understand its Operand Class. |

The base fetchmusts encoding (the middle 2 bits `0xx0`) is as follows:

| Bit Pattern |             Operand               | Operand Class | Description |
| :---------: | :-------------------------------: | :-----------: | :---------: |
|     `00`    | Complex Register Addressing (CRA) |    Memory     | Same as the one from the operand template. |
|     `01`    |   Context Relevant Encoding #2    |      TBD      | Refer to [this section](#context-relevant-uod-encodings). |
|     `10`    |    Absolute Memory Addressing     |    Memory     | Same as the one from the operand template. |
|     `11`    |   Context Relevant Encoding #3    |      TBD      | Refer to [this section](#context-relevant-uod-encodings). |

#### Context-Relevant UOD Encodings

| Number | Contextual Meaning |
| :----: | :----------------: |
|  `#1`  | When `TSZ = 8-bit`, this field encodes a byte-register ID (specifically `ho`), with OC `Register`. When `TSZ = 16-bit`, it encodes the fetchmust `Shorthand Displacement (ShD)`, with OC `Memory`. |
|  `#2`  | When `the encoding UOD is specifically OTO DB's DST UOD, and, no port-override prefix is present`, it encodes `Three-Operand Escape Sequence`, signaling that an `OTO EB (OTO Extension Byte)` will follow the OTO DB. Otherwise, it encodes `Imm8`. |
|  `#3`  | When `TSZ = 8-bit`, this field encodes the fetchmust `Shorthand Displacement (ShD)`, with OC `Memory`. When `TSZ = 16-bit`, it encodes `Imm16`. |

### OTO EB (Operand-to-Operand Extension Byte) Layout

The OTO Extension Byte is the mechanism that exposes the **destination-commit (`DC`)** and **destination-facade (`DF`)** system. It's triggered by encoding the DST UOD of an OTO DB as the base `Imm8` encoding, since such encoding would normally result in a `NIE` being raised. The high nibble of the EB denotes the `DC (Destination-Commit) UOD`, the low nibble of the EB denotes the `DF (Destination-Facade) UOD`.

| Bit Offset | Field  |
| :--------: | :----: |
|    `7-4`   | DC UOD |
|    `3-0`   | DF UOD |

Do not forget the fact that the `SRC UOD` within the OTO DB still continues to encode the source operand.  
The DC UOD must satisfy the named requirement **CommittableTo**. The DF UOD does not need to satisfy it as it won't receive stores.
