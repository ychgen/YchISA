## Opcode(s) 27-2D; Instruction `lea - Load Effective Address`

Calculates the effective address of a memory operand and loads it into the destination operand. No memory read/write ever occurs.

### Instruction Encoding

|  Opcode  |  Instruction   | O/E | TSZ |               Description               |
| :------: | :------------: | :-: | :-: | :-------------------------------------: |
|    29    | `lea dst, src` | OTO |  N  | Load effective address of src into dst. |
| 2A+r[^1] | `lea r16, shd` | OMd |  N  | Load effective address of shd into r16. |
| 2D+r[^1] | `lea r16, cra` | OMc |  N  | Load effective address of cra into r16. |

[^1]: `+r` encodings only encode, in ascending opcode order: { `ra`, `rb`, `rp` }.

### Instruction Operand Encoding

| O/E |   Operand #1   |   Operand #2   |
| :-: | :------------: | :------------: |
| OTO | OTO DB:DST[^2] | OTO DB:SRC[^3] |
| OMd |  opcode + Rid  |      ShD       |
| OMc |  opcode + Rid  |     CRA DB     |

[^2]: Operand must satisfy `CommittableTo`.  
[^3]: Operand must be of Operand Class (OC): Memory.

### Flags Affected

None.

### Specialized Exceptions

#### Nonsensical Instruction Exception (NIE)

If the source operand is not of Operand Class (OC): Memory.

### Blacklists

For O/E = OTO:
- Three-Operand Escape

Prefixes (all opcodes):
-  F  (Preserve Status Flags)
-  T  (Transaction Size Alter)
- SOP (Source Port Override)

Prefixes (O/E = `OM` opcodes):
- DOP (Destination Port Override)

### Pseudocode

```
dst := effective_address(src);
```
