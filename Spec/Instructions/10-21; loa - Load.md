## Opcode(s) 10-21; Instruction `loa - Load`

THE data movement instruction of the Ych ISA. Loads a copy of source into destination.

### Instruction Encoding

|  Opcode  |     Instruction     | O/E | TSZ |             Description             |
| :------: | :-----------------: | :-: | :-: | :---------------------------------: |
|    10    |   `loa dst, src`    | OTO |  N  | Load src into dst.                  |
|    11    | `loa byte dst, src` | OTO |  8  | Load src into dst.                  |
| 12+r[^1] |   `loa r16, imm8`   | OI  |  N  | Load sign-extended imm8 into r16.   |
|   18+r   |   `loa r8, imm8`    | OI  |  8  | Load imm8 into r8.                  |
|    20    |  `loa shd8, imm8`   | MdI |  8  | Load imm8 to shd8.                  |
|    21    |  `loa shd16, imm8`  | MdI |  N  | Load sign-extended imm8 into shd16. |

[^1]: Opcode `12+r16` lacks `rs` encoding.

### Instruction Operand Encoding

| O/E |   Operand #1   | Operand #2 |
| :-: | :------------: | :--------: |
| OTO | OTO DB:DST[^2] | OTO DB:SRC |
| OI  |  Opcode + Rid  |    Imm8    |
| MdI |      ShD       |    Imm8    |

[^2]: Operand must satisfy `CommittableTo`.  

### Flags Affected

None.

### Specialized Exceptions

None.

### Blacklists

For `O/E = OTO`:
- Three-Operand Escape

Prefixes (all opcodes):
- F (Preserve Status Flags)
- T (Transaction Size Alter)

Prefixes (O/E = `OI` opcodes):
- DOP (Destination Port Override)
- SOP (Source Port Override)

### Pseudocode

```
dst := src;
```
