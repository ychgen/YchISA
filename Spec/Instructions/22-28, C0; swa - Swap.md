## Opcode(s) 21-28, C0; Instruction `swa - Swap`

Swaps/exchanges two operands. This means that after the instruction is executed, both will store each other's initial values before the exchange.

### Instruction Encoding

|   Opcode   |     Instruction     | O/E | TSZ |    Description     |
| :--------: | :-----------------: | :-: | :-: | :----------------: |
|     22     |   `swa dst, src`    | OTO |  N  | Swap dst with src. |
|     23     | `swa byte dst, src` | OTO |  8  | Swap dst with src. |
|  24+r[^1]  |    `swa ra, r16`    |  O  |  N  | Swap ra with r16.  |
|  24+r[^1]  |    `swa r16, ra`    |  O  |  N  | Swap r16 with ra.  |
|     C0     |    `swa rb, rp`     | ZO  |  N  | Swap rb with rp.   |
|     C0     |    `swa rp, rb`     | ZO  |  N  | Swap rp with rb.   |

[^1]: Opcode `22+r` lacks `ra` and `rs` encodings.

### Instruction Operand Encoding

| O/E |     Operand #1       |      Operand #2      |
| :-: | :------------------: | :------------------: |
| OTO |    OTO DB:DST[^2]    |    OTO DB:SRC[^2]    |
|  O  |         ra           | Opcode + Rid (N)[^1] |
|  O  | Opcode + Rid (N)[^1] |          ra          |
| ZO  |         rb           |          rp          |
| ZO  |         rp           |          rb          |

[^2]: Operand must satisfy `CommittableTo`.  

### Flags Affected

None.

### Specialized Exceptions

None.

### Blacklists

For O/E = OTO:
- Three-Operand Escape

Prefixes (all opcodes):
- F (Preserve Status Flags)

Prefixes (O/E = `O` and `ZO` opcodes):
-  T  (Transaction Size Alter)
- DOP (Destination Port Override)
- SOP (Source Port Override)

### Pseudocode

```
tmp := dst;
dst := src;
src := tmp;
```
