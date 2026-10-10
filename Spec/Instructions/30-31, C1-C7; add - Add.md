## Opcode(s) 30-31; Instruction `add - Add`

Store the sum of two operands, while also updating the PSW.

### Instruction Encoding

|  Opcode  |      Instruction       | O/E  | TSZ |                   Description                    |
| :------: | :--------------------: | :--: | :-: | :----------------------------------------------: |
|    30    |     `add dst, src`     | OTO  |  N  | Load sum of dst and src into dst.                |
| 30 OTO.E |   `add dc, df, src`    | TOTO |  N  | Load sum of df and src into dc.                  |
|    31    |  `add byte dst, src`   | OTO  |  8  | Load sum of dst and src into dst.                |
| 31 OTO.E | `add byte dc, df, src` | TOTO |  8  | Load sum of df and src into dc.                  |
|   C1+r   |     `add r16, imm8`    |  OI  |  N  | Load sum of r16 and sign-extended imm8 into r16. |

### Instruction Operand Encoding

| O/E  |   Operand #1   | Operand #2 | Operand #3 |
| :--: | :------------: | :--------: | :--------: |
| OTO  | OTO DB:DST[^2] | OTO DB:SRC |    N/A     |
| TOTO | OTO EB:DC[^2]  | OTO EB:DF  | OTO DB:SRC |
|  OI  |  Opcode + Rid  |    Imm8    |    N/A     |

[^2]: Operand must satisfy `CommittableTo`.  

### Flags Affected

`ZF`: Set if the sum was equal to zero, clear otherwise.
`CF`: Set if carry on the TS-logical most-significant-bit occurred, clear otherwise.
`OF`: Set if signed overflow occurred, i.e. both operands had the same sign but the sum sign bit differed; clear otherwise.
`SF`: Set to the TS-logical most-significant-bit of the sum.

### Specialized Exceptions

None.

### Blacklists

For `O/E = (T)OTO`:
- Three-Operand Escape

Prefixes (`TSZ = 8` opcodes):
- T (Transaction Size Alter)

### Pseudocode

```
dst := dst + src;
```
