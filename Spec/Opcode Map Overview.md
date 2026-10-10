# Opcode Map Overview

## Primary Opcode Space Partitioning

The primary opcode space is partitioned as such:

| Byte/Range |          Group          |
| :--------: | :---------------------: |
|   00-0F    | Prefixes                |
|   10-1F    | LOAs                    |
|   20-2F    | Data Primitives         |
|   30-4F    | Ordinary Arithmetic     |
|   50-5F    | General Control Flow    |
|   60-6F    | Extra Data Primitives   |
|   70-8F    | Branching               |
|   90-9F    | Stack Pushing & Popping |
|   A0-AF    | Increment & Decrement   |
|   B0-BF    | Bit Primitives          |
|   C0-DF    | Density Improvisations  |
|   E0-EF    | Megainstructions        |
|   F0-FF    | Processor Control       |

Do note that within the ranges, `55` & `AA` are excluded as they are escape codes to the secondary and tertiary opcode spaces respectively.  
As of now, the Ych16 ISA does not utilize those opcode spaces, and if the decoder fetched `55` or `AA` for an opcode, it will immediately raise `Nonsensical Instruction Exception (NIE)` since nothing meaningful can potentially follow. They are reserved for future expansion.  
Do note that the groups are mostly suggestions. If things get tight, you might see some weird stuff!
